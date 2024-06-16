# Getting started with Proxy Mode

## Contents

## What is the proxy mode?

Spider also offers a proxy front-end to the API. This can make integration with third-party tools easier. The Proxy mode only changes the way you access Spider. The Spider API will then handle requests just like any standard request.

Request cost, return code and default parameters will be the same as a standard no-proxy request.

We recommend disabling JavaScript rendering in proxy mode, which is enabled by default. The following credentials and configurations are used to access the proxy mode:

- **HTTP address**: `proxy.spider.cloud:8888`
- **HTTPS address**: `proxy.spider.cloud:8889`
- **Username**: `YOUR-API-KEY`
- **Password**: `PARAMETERS`

Important: Replace `PARAMETERS` with our supported API parameters. If you don't know what to use, you can begin by using `render_js=False`. If you want to use multiple parameters, use `&` as a delimiter, example: `render_js=False&premium_proxy=True`.

As an alternative, you can use URLs like the following:

```json
{
	"url": "http://proxy.spider.cloud:8888",
	"username": "YOUR-API-KEY",
	"password": "render_js=False&premium_proxy=True"
}
```

## Spider Proxy Features

- **Premium proxy rotations**: no more headaches dealing with IP blocks
- **Cost-effective**: 1 credit per base request and two credits for premium proxies.
- **Full concurrency**: crawl thousands of pages in seconds, yes that isn't a typo!
- **Caching**: repeated page crawls ensures a speed boost in your crawls
- **Optimal response format**: Get clean and formatted markdown, HTML, or text for LLM and AI agents
- **Avoid anti-bot detection**: measures that further lower the chances of crawls being blocked
- [And many more](/docs/api)

## HTTP Proxies built to scale

At the time of writing, we now have http and https proxying capabilities to leverage Spider to gather data.

## Proxy Usage

Getting started with the proxy is simple and straightforward. After you get your [secret key](/api-keys)
you can access our instance directly.

```py
import requests, os, logging

# Set debug level logging
logging.basicConfig(level=logging.DEBUG)

# Proxy configuration
proxies = {
    'http': f"http://{os.getenv('SPIDER_API_KEY')}:request=Raw&premium_proxy=False@proxy.spider.cloud:8888",
    'https': f"https://{os.getenv('SPIDER_API_KEY')}:request=Raw&premium_proxy=False@proxy.spider.cloud:8889"
}

# Function to make a request through the proxy
def get_via_proxy(url):
    try:
        response = requests.get(url, proxies=proxies)
        response.raise_for_status()
        logging.info('Response HTTP Status Code: ', response.status_code)
        logging.info('Response HTTP Response Body: ', response.content)
        return response.text
    except requests.exceptions.RequestException as e:
        logging.error(f"Error: {e}")
        return None

# Example usage
if __name__ == "__main__":
     get_via_proxy("https://www.choosealicense.com")
     get_via_proxy("https://www.choosealicense.com/community")
```

## Request Cost Structure

| Request Type        | Cost (Credits) |
| ------------------- | -------------- |
| Base request (HTTP) | 1 credit       |
| Premium proxies     | 2 credits      |
| Chrome              | 4 credits      |
| Smart-mode          | 1 - 4 credits  |


### Coming soon

Some of the params and socks5 are not available at the time of writing.

1. JS rendering.
1. Transforming to markdown etc.
1. Readability.
