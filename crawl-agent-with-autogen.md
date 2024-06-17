# Scrape & Crawl Agent with Microsoft's Autogen

This guide will show you how to set up an [Autogen](https://www.microsoft.com/en-us/research/project/autogen/) agent to scrape and crawl any website using the Spider API.

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

## Setup Spider & Autogen

Getting started with the API is simple and straightforward. After you get your [secret key](https://spider.cloud/api-keys) you can follow this guide. We won't rehash the full setup guide for Spider here, but if you want to use the API directly, you can check out the [Spider API Guide](https://spider.cloud/guides/spider-api) to learn more. Let's move on.

Install the Spider Python client library and autogen:

```bash
pip install spider-client pyautogen
```

Now we need to setup the Autogen LLM configuration.

```python
import os

config_list = [
    {"model": "gpt-4o", "api_key": os.getenv("OPENAI_API_KEY")},
]
```

And we need to set the Spider API key:

```python
spider_api_key = os.getenv("SPIDER_API_KEY")
```

## Creating Scrape & Crawl Functions

We first need to import spider so that we can call the API to be able to scrape and crawl.

```python
from spider import Spider
```

### Defining functions for the agents

Now we need to define the scrape and crawl function that the agent will call. We will use the python Spider SDK for this and set the default `return_format` to `markdown` to retrieve LLM-ready data.

```python
from typing_extensions import Annotated
from typing import List, Dict, Any

def scrape_page(url: Annotated[str, "The URL of the web page to scrape"], params: Annotated[dict, "Dictionary of additional params."] = None) -> Annotated[Dict[str, Any], "Scraped content"]:
    # Initialize the Spider client with your API key, if no api key is specified it looks for SPIDER_API_KEY in your environment variables
    client = Spider(spider_api_key)

    if params == None:
        params = {
            "return_format": "markdown"
        }

    scraped_data = client.scrape_url(url, params)
    return scraped_data[0]

def crawl_page(url: Annotated[str, "The url of the domain to be crawled"], params: Annotated[dict, "Dictionary of additional params."] = None) -> Annotated[List[Dict[str, Any]], "Scraped content"]:
    # Initialize the Spider client with your API key, if no api key is specified it looks for SPIDER_API_KEY in your environment variables
    client = Spider(spider_api_key)

    if params == None:
        params = {
            "return_format": "markdown"
        }

    crawled_data = client.crawl_url(url, params)
    return crawled_data
```

Now that we have the functions defined, we need to create the scrape & crawl agents, and let them know that they can use the functions to scrape & crawl any website.

Here is also when we use the `config_list` we defined at the top of this guide.

```python
from autogen import ConversableAgent

# Create web scraper agent.
scraper_agent = ConversableAgent(
    "WebScraper",
    llm_config={"config_list": config_list},
    system_message="You are a web scraper and you can scrape any web page to retrieve its contents."
    "Returns 'TERMINATE' when the scraping is done.",
)

# Create web crawler agent.
crawler_agent = ConversableAgent(
    "WebCrawler",
    llm_config={"config_list": config_list},
    system_message="You are a web crawler and you can crawl any page with deeper crawling following subpages."
    "Returns 'TERMINATE' when the scraping is done.",
)
```

### How do we tell the agents to do things?

To be able to chat and make these agents actually do something, we need a `UserProxyAgent` that can communicate with the other agents:

You can read more about the [UserProxyAgent here](https://microsoft.github.io/autogen/docs/reference/agentchat/user_proxy_agent/).

```python
user_proxy_agent = ConversableAgent(
    "UserProxy",
    llm_config=False,  # No LLM for this agent.
    human_input_mode="NEVER",
    code_execution_config=False,  # No code execution for this agent.
    is_termination_msg=lambda x: x.get("content", "") is not None and "terminate" in x["content"].lower(),
    default_auto_reply="Please continue if not finished, otherwise return 'TERMINATE'.",
)
```

### Registering the functions

Now when we have the agents and the `user_proxy_agent` we can officially register the functions with the correct agents, and the agents with the user_proxy_agent using `register_function` provided from Autogen.

```python
from autogen import register_function

register_function(
    scrape_page,
    caller=scraper_agent,
    executor=user_proxy_agent,
    name="scrape_page",
    description="Scrape a web page and return the content.",
)

register_function(
    crawl_page,
    caller=crawler_agent,
    executor=user_proxy_agent,
    name="crawl_page",
    description="Crawl an entire domain, following subpages and return the content.",
)
```

Now we have officially linked all the agents together and can try talking to user_proxy_agent.

## Using the agents

We can start the conversation with user_proxy_agent and say that we either want to crawl or scrape a specific website.

Then we can summarize the scraped and crawled page with Autogen's built in summary_method. We use reflection_with_llm to create a summary based on the conversation history, AKA the scraped or crawled content.

```python
# Scrape page
scraped_chat_result = user_proxy_agent.initiate_chat(
    scraper_agent,
    message="Can you scrape william-espegren.com for me?",
    summary_method="reflection_with_llm",
    summary_args={
        "summary_prompt": """Summarize the scraped content"""
    },
)

# Crawl page
crawled_chat_result = user_proxy_agent.initiate_chat(
    crawler_agent,
    message="Can you crawl william-espegren.com for me, I want the whole domains information?",
    summary_method="reflection_with_llm",
    summary_args={
        "summary_prompt": """Summarize the crawled content"""
    },
)
```

The output is stored in the summary:

```python
print(scraped_chat_result.summary)
print(crawled_chat_result.summary)
```

## Full code

Now we have two agents: one that scrapes a page and one that crawls a page following subpages. These two agents can you use in combination with your other Autogen agents.

```python
import os
from spider import Spider
from typing_extensions import Annotated
from typing import List, Dict, Any
from autogen import ConversableAgent
from autogen import register_function

config_list = [
    {"model": "gpt-4o", "api_key": os.getenv("OPENAI_API_KEY")},
]

spider_api_key = os.getenv("SPIDER_API_KEY")

def scrape_page(url: Annotated[str, "The URL of the web page to scrape"], params: Annotated[dict, "Dictionary of additional params."] = None) -> Annotated[Dict[str, Any], "Scraped content"]:
    # Initialize the Spider client with your API key, if no api key is specified it looks for SPIDER_API_KEY in your environment variables
    client = Spider(spider_api_key)

    if params == None:
        params = {
            "return_format": "markdown"
        }

    scraped_data = client.scrape_url(url, params)
    return scraped_data[0]

def crawl_page(url: Annotated[str, "The url of the domain to be crawled"], params: Annotated[dict, "Dictionary of additional params."] = None) -> Annotated[List[Dict[str, Any]], "Scraped content"]:
    # Initialize the Spider client with your API key, if no api key is specified it looks for SPIDER_API_KEY in your environment variables
    client = Spider(spider_api_key)

    if params == None:
        params = {
            "return_format": "markdown"
        }

    crawled_data = client.crawl_url(url, params)
    return crawled_data

# Create web scraper agent.
scraper_agent = ConversableAgent(
    "WebScraper",
    llm_config={"config_list": config_list},
    system_message="You are a web scraper and you can scrape any web page to retrieve its contents."
    "Returns 'TERMINATE' when the scraping is done.",
)

# Create web crawler agent.
crawler_agent = ConversableAgent(
    "WebCrawler",
    llm_config={"config_list": config_list},
    system_message="You are a web crawler and you can crawl any page with deeper crawling following subpages."
    "Returns 'TERMINATE' when the scraping is done.",
)

user_proxy_agent = ConversableAgent(
    "UserProxy",
    llm_config=False,  # No LLM for this agent.
    human_input_mode="NEVER",
    code_execution_config=False,  # No code execution for this agent.
    is_termination_msg=lambda x: x.get("content", "") is not None and "terminate" in x["content"].lower(),
    default_auto_reply="Please continue if not finished, otherwise return 'TERMINATE'.",
)

register_function(
    scrape_page,
    caller=scraper_agent,
    executor=user_proxy_agent,
    name="scrape_page",
    description="Scrape a web page and return the content.",
)

register_function(
    crawl_page,
    caller=crawler_agent,
    executor=user_proxy_agent,
    name="crawl_page",
    description="Crawl an entire domain, following subpages and return the content.",
)

# Scrape page
scraped_chat_result = user_proxy_agent.initiate_chat(
    scraper_agent,
    message="Can you scrape william-espegren.com for me?",
    summary_method="reflection_with_llm",
    summary_args={
        "summary_prompt": """Summarize the scraped content"""
    },
)

print(scraped_chat_result.summary)
```

If you liked this guide, consider checking out me and Spider on Twitter:

- **Author Twitter:** [WilliamEspegren](https://x.com/WilliamEspegren)
- **Spider Twitter:** [spider_rust](https://x.com/spider_rust)