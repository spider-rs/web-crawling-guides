## Spider API Features

- **Premium proxy rotations**: no more headaches dealing with IP blocks
- **Cost-effective**: crawl many pages for a fraction of the cost of what other services charge
- **Full concurrency**: crawl thousands of pages in seconds, yes that isn't a typo!
- **Smart mode**: ensure fast crawling while considering javascript rendered pages into account using headless Chrome
- **Caching**: repeated page crawls ensures a speed boost in your crawls
- **Optimal response format**: Get clean and formatted markdown, HTML, or text for LLM and AI agents
- **Scrape with AI (Beta)**: do custom browser scripting and data extraction using the latest AI models
- **Avoid anti-bot detection**: measures that further lowers the chances of crawls being blocked
- [And many more](/docs/api)


## API built to scale

Welcome to the fastest web crawler API. We want you to experience the full potential of our platform, which is why we have designed our API to be highly scalable and efficient. 

Our platform is designed to effortlessly manage thousands of requests per second, thanks to our elastically scalable system architecture and the Open-Source [spider](https://github.com/spider-rs/spider) project. We deliver consistent latency times ensuring processing for all responses. 

For an in-depth understanding of the request parameters supported, we invite you to explore our comprehensive API documentation. At present, we do not provide client-side libraries, as our API has been crafted with simplicity in mind for straightforward usage. However, we are open to expanding our offerings in the future to enhance user convenience.

Dive into our [documentation](/docs/api) to get started and unleash the full potential of our web crawler today.

## API usage

Getting started with the API is simple and straight forward. After you get your [secret key](/api-keys)
you can access our instance directly. Or if you prefer, you can use our client SDK libraries for [python](https://github.com/spider-rs/spider-clients/tree/main/python) and [javascript](https://github.com/spider-rs/spider-clients/tree/main/javascript). The crawler is highly configurable through the params to fit all needs and use cases when using the API directly or client libraries.

## Use Spider in LangChain or LlamaIndex
### LangChain
- [Documentation](https://python.langchain.com/v0.1/docs/integrations/document_loaders/spider/) using Spider as a data loader (Python)
- Javascript coming soon!

### LlamaIndex
- [Documentation](https://docs.llamaindex.ai/en/stable/examples/data_connectors/WebPageDemo/?h=spider#using-spider-reader) for using Spider as a web page reader

## Crawling one page

Most cases you probally just want to crawl one page. Even if you only need one page, our system performs fast enough to lead the race.
The most straight forward way to make sure you only crawl a single page is to set the [budget limit](/account/settings) with a wild card value or `*` to 1.
You can also pass in the param `limit` in the JSON body with the limit of pages.

## Crawling multiple pages

When you crawl multiple pages, the concurrency horsepower of the spider kicks in. You might wonder why and how one request may take (x)ms to come back, and 100 requests take about the same time! That's because the built-in isolated concurrency allows for crawling thousands to millions of pages in no time. It's the only current solution that can handle large websites with over 100k pages within a minute or two (sometimes even in a blink or two). By default, we do not add any limits to crawls unless specified.


## Optimized response format for large language models (LLM) and AI agents

Get the response in markdown that is clean, easy to parse, and save on token cost when using LLMs.
```py

json_data = {""return_format":"markdown","url":"http://www.example.com", "limit": 5}

response = requests.post('https://api.spider.cloud/v1/crawl', 
  headers=headers, 
  json=json_data)
```

Or if you prefer, you can get the response in `raw` HTML format or `text` only.

## Planet scale crawling

If you plan on processing crawls that have over 200 pages, we recommend streaming the request from the client instead of parsing the entire payload once finished. We have an example of this with Python on the API docs page, also shown below.

```py
import requests, os, json

headers = {
    'Authorization': os.environ["SPIDER_API_KEY"],
    'Content-Type': 'application/json',
}

json_data = {"limit":250,"url":"http://www.example.com"}

response = requests.post('https://api.spider.cloud/v1/crawl', 
  headers=headers, 
  json=json_data,
  stream=True)

for line in response.iter_lines():
  if line:
      print(json.loads(line))
```

### Automatic configuration

Spider handles automatic concurrency handling, proxy rotations (if enabled), anti-bot measures, and more. 

## Get started with using the API
1. [Add credits](/credits/new) to your account balance. The more credits you have or usage available allows for a higher concurrency limit.
2. [Create](/api-keys) an API key

Thanks for using Spider! We are excited to see what you build with our API. If you have any questions or need help, please contact us through the feedback form.