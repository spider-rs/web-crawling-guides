# LangChain + Groq + Spider = 🚀 (Integration Guide)

This guide will show you the power of combining LangChain, with the fastest inference available using Groq and the fastest crawler API available. We will crawl multiple URLs and then pass the contents to an LLM to summarize using the Meta's new [LLama 3](https://llama.meta.com/llama3/) model.

## Setup Groq

Get Groq setup and running in a few minutes with the following steps:

1. Create an API Key [here](https://console.groq.com/keys)

2. Install Groq and setup the api key in your project as an environment variable. Simple approach that prevents you from hardcoding the key in your code.

```bash
pip install groq
```

In your terminal:

```bash
export GROQ_API_KEY=<your-api-key-here>
```

Alternatively, you can use the `dotenv` package to load the environment variables from a `.env` file. Create a `.env` file in your project root and add the following:

```bash
GROQ_API_KEY=<your-api-key-here>
```

Then in your Python code:

```py
from dotenv import load_dotenv
import os

load_dotenv()

client = Groq(
    api_key=os.environ.get("GROQ_API_KEY"),
)
```

3. Test groq and see if things are working correctly:

```python
import os

from groq import Groq

client = Groq(
    api_key=os.environ.get("GROQ_API_KEY"),
)

chat_completion = client.chat.completions.create(
    messages=[
        {
            "role": "user",
            "content": "What are large language models?",
        }
    ],
    model="llama3-8b-8192",
)
```

We'll be using the Llama 3 8B model for this example–very fast! This model should give us about 800 tokens per second or more. The larger 70B model will give you around 150 tokens per second.

## Setup Spider

Getting started with the API is simple and straight forward. After you get your [secret key](/api-keys)
you can use the Spider LangChain document loader. We won't rehash the full setup guide for Spider here if you want to use the API directly, you can check out the [Spider API Guide](spider-api) to learn more. Let's move on.

Install the Spider Python client library:

```bash
pip install spider-client
```

Setup the api key in the env file:

```bash
SPIDER_API_KEY=<your-api-key-here>
```

Then import the `SpiderLoader` from the document loaders module:

```python
from langchain_community.document_loaders import SpiderLoader
```

Let's setup the Spider API for our example use case:

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

Set the mode to `crawl` and we use the `return_format` parameter to specify we want to markdown content. The rest of the parameters are optional.

### Reminder

Spider handles automatic concurrency handling and ip rotation to make it simple to scrape multiple urls at once.
The more credits you have or usage available allows for a higher concurrency limit. So make sure you have enough credits if you choose to cawl more than one page.

For now, we'll turn off the proxy and move on to setting up LangChain.

## Setup LangChain

Install LangChain if you haven't yet:

```bash
pip install langchain
```

Let's install the Groq LangChain package

```bash
pip install langchain-groq
```

[LangChain](https://www.langchain.com/) is a powerful LLM orchestration API framework, but in this example we'll use it more simply to put together our prompt and run a chain. To see LangChain's features and capabilities, check out their API documentation [here](https://python.langchain.com/).

Let's setup LangChain with a simple chain using the chat prompt template:

```python
from langchain_core.prompts import ChatPromptTemplate
markdown_content = "Ancana: Marketplace to buy managed vacation   homes through fractional ownership | Y Combinator...."
system = "You are a helpful assistant. Please summarize this company in bullet points."
prompt = ChatPromptTemplate.from_messages(
    [("system", system), ("human", "{markdown_text}")]
)

chain = prompt | chat

urls = [
    "https://www.ycombinator.com/companies/airbnb",
    "https://www.ycombinator.com/companies/ancana",
]

summary = chain.invoke({"markdown_text": markdown_content})
print(summary.content)
```

And the results should be something like this:

```markdown
## output

**Ancana:**

- Founded in 2019
- Marketplace for buying and owning fractional shares of vacation homes
- Allows users to purchase a share of a property and split expenses with other co-owners
- Offers features like property management, furnishing, and maintenance
- Founded by Andres Barrios and Ryan Black
```

Great, tt's looking good now! The goal now is we want to crawl two startup page descriptions from Y-Combinator's site, summarize them and have the LLM tell me the difference between the two.

But first we need to combine the markdown content from both companies and format it so that the LLM can easily understand it.

```py
def concat_markdown_content(markdown_content):
    final_content = ""
    separator = "=" * 40

    for content_dict in markdown_content:
        url = content_dict["url"]
        markdown_text = content_dict["markdown_text"]
        title = content_dict["title"]
        final_content += f"{separator}\nURL: {url}\nPage Title: {title}\nMarkdown:\n{separator}\n{markdown_text}\n"

    return final_content
```

By adding a demarcation and some metadata, the LLM should have enough context to summarize and compare the two companies.

Let's also update the system prompt to help us achieve our goal.

```python
system = "You are a helpful assistant. Please summarize the markdown content for Ancana and AirBNB provided in the context in bullet points separately. And then provide a summary of how they are similar and different."
```

And that's all! Check out the full code and we'll update this guide to link out to a notebook, so you can easily play with it.

## Full code

```python
from langchain_community.document_loaders import SpiderLoader
from langchain_core.prompts import ChatPromptTemplate
from langchain_groq import ChatGroq
from dotenv import load_dotenv
import os
import time

load_dotenv()

chat = ChatGroq(
    temperature=0,
    groq_api_key=os.environ.get("GROQ_API_KEY"),
    model_name="llama3-70b-8192",
)


def load_markdown_from_urls(url):
    loader = SpiderLoader(
        url=url,
        mode="crawl",
        params={
            "return_format": "markdown",
            "proxy_enabled": False,
            "request": "http",
            "request_timeout": 60,
            "limit": 1,
            "cache": False,
        },
    )
    data = loader.load()

    if data:
        return data
    else:
        return None


def concat_markdown_content(markdown_content):
    final_content = ""
    separator = "=" * 40

    for content_dict in markdown_content:
        url = content_dict["url"]
        markdown_text = content_dict["markdown_text"]
        title = content_dict["title"]
        final_content += f"{separator}\nURL: {url}\nPage Title: {title}\nMarkdown:\n{separator}\n{markdown_text}\n"

    return final_content


system = "You are a helpful assistant. Please summarize the markdown content for Ancana and AirBNB provided in the context in bullet points separately. And then provide a summary of how they are similar and different."
prompt = ChatPromptTemplate.from_messages(
    [("system", system), ("human", "{markdown_text}")]
)

chain = prompt | chat

start_time = time.time()
urls = [
    "https://www.ycombinator.com/companies/airbnb",
    "https://www.ycombinator.com/companies/ancana",
]

url_join = ",".join(urls)
markdown_contents = load_markdown_from_urls(url_join)
all_contents = []

for content in markdown_contents:
    if content.page_content:
        all_contents.append(
            {
                "markdown_text": content.page_content,
                "url": content.metadata["url"],
                "title": content.metadata["title"],
            }
        )

    concatenated_markdown = concat_markdown_content(all_contents)

summary = chain.invoke({"markdown_text": concatenated_markdown})
end_time = time.time()
elapsed_time = end_time - start_time
print(f"Elapsed time: {elapsed_time:.2f} seconds")
print(summary.content)
```

```markdown
# Output

Here are the bullet points summarizing Ancana:

**Ancana**

- Marketplace to buy managed vacation homes through fractional ownership
- Allows individuals to own a share of a vacation home (e.g. 1/4 or 1/8) and split expenses with other co-owners
- Properties are furnished and managed by Ancana, with costs split amongst co-owners
- Owners have access to the property for a set period of time (e.g. 3 months per year)
- Founded in 2019, based in Mérida, Mexico
- Team size: 7
- Founders: Andres Barrios, Ryan Black

And here are the bullet points summarizing Airbnb:

**Airbnb:**

- Online marketplace for booking unique accommodations around the world
- Founded in 2008 and based in San Francisco, California
- Allows users to list and discover unique spaces, from apartments to castles
- Offers a trusted community marketplace for people to monetize their extra space
- Has over 33,000 cities and 192 countries listed on the platform
- Team size: 6,132

**Similarities:**

- Both Ancana and Airbnb operate in the travel and accommodation space
- Both platforms allow users to book and stay in unique properties
- Both companies focus on providing a trusted and community-driven marketplace for users

**Differences:**

- Ancana focuses on fractional ownership of vacation homes, while Airbnb is a booking platform for short-term rentals
- Ancana properties are fully managed and furnished, while Airbnb listings can vary in terms of amenities and services
- Ancana is a newer company with a smaller team size, while Airbnb is a more established company with a larger team and global presence
```

When benchmarking this example we got an elapsed time of around 7.58 seconds (Cache disabled) total.

So now we got a full working code that demonstrates the power of Groq and Spider, coupled with LangChain to format our prompts together. Thanks for following along! Stay tuned for more guides.
