# Automated Cold Email Outreach Using Spider

This guide will show you how to automate the process of cold email outreach by extracting email content, identifying the company behind the email, searching for their website, and crafting a personalized email using the LLM-ready data returned from the website by Spider.

## Retrieve Email

For this guide, we will not cover how to get the contents of the email, as it varies between different services. Instead, we will have a variable with the email content in it.

```python
email = '''
Thank you for your email Gilbert,

I have looked into YourBusinessName and it seems to suit some of our customers' requests, but not really so many that make it profitable for us to invest time and money integrating it into our current services. If you have any use cases in mind that suit our company, I might propose an idea to the others.

Best,
Matilda

SEO expert at Spider.cloud
'''
```

## Setup OpenAI

Get OpenAI setup and running in a few minutes with the following steps:

1. Create an account and get an API Key on [OpenAI](https://openai.com/).

2. Install OpenAI and set up the API key in your project as an environment variable. This approach prevents you from hardcoding the key in your code.

```bash
pip install openai
```

In your terminal:

```bash
export OPENAI_API_KEY=<your-api-key-here>
```

Alternatively, you can use the `dotenv` package to load the environment variables from a `.env` file. Create a `.env` file in your project root and add the following:

```bash
OPENAI_API_KEY=<your-api-key-here>
```

Then, in your Python code:

```python
from dotenv import load_dotenv
from openai import OpenAI
import os

load_dotenv()

client = OpenAI(
    api_key=os.environ.get("OPENAI_API_KEY"),
)
```

3. Test OpenAI to see if things are working correctly:

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ.get("OPENAI_API_KEY"),
)

chat_completion = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {
            "role": "user",
            "content": "What are large language models?",
        }
    ]
)
```

## Setup Spider & Langchain

Getting started with the API is simple and straightforward. After you get your [secret key](https://spider.cloud/api-keys), you can use the Spider LangChain document loader. We won't rehash the full setup guide for Spider here, but if you want to use the API directly, you can check out the [Spider API Guide](https://spider.cloud/guides/spider-api) to learn more. Let's move on.

Install the Spider Python client library and langChain:

```bash
pip install spider-client langchain langchain-community
```

Then import the `SpiderLoader` from the document loaders module:

```python
from langchain_community.document_loaders import SpiderLoader
```

Let's set up the Spider API for our example use case:

```python
def load_markdown_from_url(urls):
    loader = SpiderLoader(
        url=urls,
        mode="crawl",
        params={
            "return_format": "markdown",
            "proxy_enabled": False,
            "request": "http",
            "request_timeout": 60,
            "limit": 1,
        },
    )
    data = loader.load()
```

Set the mode to `crawl` and use the `return_format` parameter to specify we want markdown content. The rest of the parameters are optional.

### Reminder

Spider handles automatic concurrency handling and IP rotation to make it simple to scrape multiple URLs at once. The more credits you have or usage available allows for a higher concurrency limit. Make sure you have enough credits if you choose to crawl more than one page.

For now, we'll turn off the proxy and move on to setting up LangChain.

## Puzzling the pieces together

Now that we have everything installed and working, we should start with connecting the different pieces together.

First we need to extract the company name from the email:

```python
import os
from openai import OpenAI

email_content = '''
Thank you for your email Gilbert,

I have looked into yourAutomatedCRM and it seems to suit some of our customers' requests, but not really so many that makes it profitable for us to invest time and money integrating it into our current services. If you have any use cases in mind that suit our company, I might be able to propose an idea to the others.

Best,
Matilda

SEO expert at Spider.cloud
'''

# Initialize OpenAI client
client = OpenAI(
    api_key=os.environ.get("OPENAI_API_KEY")
)

def extract_company_name(email):
    # Define messages
    messages = [{"role": "user", "content": f'Extract the company name and return ONLY the company name from the sender of this email: """{email_content}"""'}]

    # Call OpenAI API
    completion = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=messages,
    )

    return completion.choices[0].message.content

company_name = extract_company_name(email_content)
print(company_name)
```

### Finding the company's official website

By using Spider's built in AI scraping tools, we can specify our own prompt in our Spider API request.

"Return the official website of the company for `company-name`" on a bing search suits really well for this guide, since we want the url for the company's website.'

``` python
import requests, os

headers = {
    'Authorization': os.environ["SPIDER_API_KEY"],
    'Content-Type': 'application/json',
}

json_data = {
    "limit":1,
    "gpt_config":{
        "prompt":f'Return the official website of the company for {company_name}',
        "model":"gpt-4o",
        "max_tokens":4096,
        "temperature":0.54,
        "top_p":0.17,
    },
    "url":"https://www.bing.com/search?q=spider.cloud"
}

response = requests.post('https://api.spider.cloud/crawl', 
  headers=headers, 
  json=json_data
)

company_url = response.json()[0]['metadata']['extracted_data']
```

## Explanation of the `gpt_config`
The gpt_config in the Spider API request specifies the configuration for the GPT model used to do actions on the scraped data. It includes parameters such as:

- prompt: The prompt provided to the model (string or a list of strings)
- model: The specific GPT model to use.
- max_tokens: The maximum number of tokens to generate.
- temperature: Controls the randomness of the output (higher values make output more random).
- top_p: Controls the diversity of the output (higher values make output more diverse).

These settings ensure that the API generates coherent and contextually appropriate responses based on the scraped data.

### Crafting super personalized email based on the websites content
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain import hub
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI
from langchain_community.document_loaders import SpiderLoader

company_url = 'https://spider.cloud'

def filter_metadata(doc):
    # Filter out or replace None values in metadata
    doc.metadata = {k: (v if v is not None else "") for k, v in doc.metadata.items()}
    return doc

def load_markdown_from_url(urls):
    loader = SpiderLoader(
        # env="your-api-key-here", # if no API key is provided it looks for SPIDER_API_KEY in env
        url=urls,
        mode="crawl",  
        params={
            "return_format": "markdown",
            "proxy_enabled": False,
            "request": "http",  
            "request_timeout": 60,
            "limit": 1,
        },
    )
    data = loader.load()
    return data

docs = load_markdown_from_url(company_url)

# Filter metadata in documents
docs = [filter_metadata(doc) for doc in docs]

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)
vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())

# Retrieve and generate using the relevant snippets of the blog.
retriever = vectorstore.as_retriever()
prompt = hub.pull("rlm/rag-prompt")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

llm = ChatOpenAI(model="gpt-4o")

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
print(rag_chain.invoke(f'Craft a super personalized email answering {company_name}, answering their response to our cold outreach campaigne. Their email: """{email_content}"""'))
```

And the results should be something like this:

```markdown
### output

Hi Matilda,

Thank you for considering AutomatedCRM. Given Spider.cloud's needs for efficient and large-scale data collection, our CRM can integrate seamlessly with tools like Spider, providing robust, high-speed data extraction and management. I'd love to discuss specific use cases where this integration could significantly enhance your current offerings.

Best,
Gilbert
```

## Why do we use `hub.pull("rlm/rag-prompt")?`

We chose hub.pull("rlm/rag-prompt") for this use case because it provides a robust and flexible template for prompt construction, specifically designed for retrieval-augmented generation (RAG) tasks. This helps in creating contextually relevant and highly personalized responses by leveraging the extracted and processed data returned from Spider.

## Complete code

That is it, now we have a fully automated email cold outreach with Spider, that responds with emails with knowledge about the companys website.

Here is the full code:

```python
import requests, os
from openai import OpenAI
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain import hub
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI
from langchain_community.document_loaders import SpiderLoader

email_content = '''
Thank you for your email Gilbert,

I have looked into yourAutomatedCRM and it seems to suit some of our customers' requests, but not really so many that makes it profitable for us to invest time and money integrating it into our current services. If you have any use cases in mind that suit our company, I might be able to propose an idea to the others.

Best,
Matilda

SEO expert at Spider.cloud
'''

# Initialize OpenAI client
client = OpenAI(
    api_key=os.environ.get("OPENAI_API_KEY")
)

def extract_company_name(email):
    # Define messages
    messages = [{"role": "user", "content": f'Extract the company name and return ONLY the company name from the sender of this email: """{email_content}"""'}]

    # Call OpenAI API
    completion = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=messages,
    )

    return completion.choices[0].message.content

company_name = extract_company_name(email_content)

headers = {
    'Authorization': os.environ["SPIDER_API_KEY"],
    'Content-Type': 'application/json',
}

json_data = {
    "limit":1,
    "gpt_config":{
        "prompt":f'Return the official website of the company for {company_name}',
        "model":"gpt-4o",
        "max_tokens":4096,
        "temperature":0.54,
        "top_p":0.17,
    },
    "url":"https://www.bing.com/search?q=spider.cloud"
}

response = requests.post('https://api.spider.cloud/crawl', 
  headers=headers, 
  json=json_data
)

company_url = response.json()[0]['metadata']['extracted_data']

def filter_metadata(doc):
    # Filter out or replace None values in metadata
    doc.metadata = {k: (v if v is not None else "") for k, v in doc.metadata.items()}
    return doc

def load_markdown_from_url(urls):
    loader = SpiderLoader(
        # env="your-api-key-here", # if no API key is provided it looks for SPIDER_API_KEY in env
        url=urls,
        mode="crawl",  
        params={
            "return_format": "markdown",
            "proxy_enabled": False,
            "request": "http",  
            "request_timeout": 60,
            "limit": 1,
        },
    )
    data = loader.load()
    return data

docs = load_markdown_from_url(company_url)

# Filter metadata in documents
docs = [filter_metadata(doc) for doc in docs]

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)
vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())

# Retrieve and generate using the relevant snippets of the blog.
retriever = vectorstore.as_retriever()
prompt = hub.pull("rlm/rag-prompt")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

llm = ChatOpenAI(model="gpt-4o")

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
print(rag_chain.invoke(f'Craft a super personalized email answering {company_name}, answering their response to our cold outreach campaigne. Their email: """{email_content}"""'))
```

If you liked this guide, consider checking out me and Spider on Twitter:
- **Author Twitter:** [WilliamEspegren](https://x.com/WilliamEspegren)
- **Spider Twitter:** [spider_rust](https://x.com/spider_rust)