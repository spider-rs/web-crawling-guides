# Stock Research Assistant Using crewAI and Spider

This guide will show you the power of using Spider with AI agents. Specifically, we're going to be using crewAI, a popular agent framework to scaffold our agent and orchestrate our research work flow. CrewAI has a great stock researcher guide on their site, so this guide is not to replace that, but to show you how to use Spider as an additional research tool.

## Install and setup crewAI

```shell
pip install crewai
```

Then, we'll install additional tool dependencies

```shell
pip install 'crewai[tools]'
```

Then setup environment variables for our OpenAI api key, and the model string we're going to use. For our simple stock research example, we'll use `gpt-4-turbo` as it has a context size of 128K, plenty for our research agents to use. An alternative is to mix the models and an assign a different model to each of the agents. You can find more information on setting different [LLM configurations in the documentation](https://docs.crewai.com/how-to/LLM-Connections/#connect-crewai-to-llms).

```py
import os
os.environ["OPENAI_API_KEY"] = "YOUR_API_KEY"
os.environ["OPENAI_MODEL_NAME"] = "gpt-4-turbo"
```

### Setup Serper.dev for Google Search

We're going to use [serper.dev](https://serper.dev/) as our search tool in crewAI. Make sure to create an account and grab your api key.

```py
import os
os.environ["SERPER_API_KEY"] = "Your Key"
from crewai_tools import SerperDevTool
search_tool = SerperDevTool()
```

Refer to crew's [documentation](https://docs.crewai.com/how-to/Creating-a-Crew-and-kick-it-off/) for more information

## Create a custom scrape tool

Before we setup our agents, we need to create a custom tool for scraping and summarizing content that the agents will be using for our report. We'll use Spider as our crawler and scraping tool. In case you're wondering, crewAI does have it's own scraping tool, but it's not as robust and fast as Spider.

### Setup Spider

Getting started with the API is simple and straight forward. After you get your [secret key](https://spider.cloud/api-keys)
you can use the Spider LangChain document loader. We won't rehash the full setup guide for Spider here if you want to use the API directly, you can check out the [Spider API Guide](https://spider.cloud/guides/spider-api) to learn more. Let's move on.

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

First, we'll code up our Spider crawler like so as a data loader for LangChain:

```py

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

```

To learn more about how to setup Spider, follow [this guide](https://spider.cloud/guides/spider-api).

Install LangChain if you haven't yet:

```bash
pip install langchain
```

[LangChain](https://www.langchain.com/) is a powerful LLM orchestration API framework, but in this example we'll use it more simply to put together our prompt and run a chain. To see LangChain's features and capabilities, check out their API documentation [here](https://python.langchain.com/).

Now, we continue to code up the rest of the summarization code, which we'll use LangChain for. We'll add the crewAI `tool` decorator so that our tool function can be utilized by our agents.

```py
from crewai_tools import tool

@tool("scrape_and_summarize")
def scrape_and_summarize(urls: List[str]) -> str:
    """Scrape website content based on one or more urls and summarize each based on the objective of the goal. Scrape up to 5 URLs at a time. Do not scrape or summarize PDF content types."""

    url_join_str = ",".join(urls)
    content_docs = load_markdown_from_urls(url_join_str)

    llm = ChatOpenAI(model="gpt-4-turbo")

    document_prompt = PromptTemplate(
        input_variables=["page_content"], template="{page_content}"
    )
    document_variable_name = "context"
    prompt = PromptTemplate.from_template(
        "Objective: Summarize this content in bullet points highlighting important insights. Be comprehensive, yet concise: {context}"
    )
    llm_chain = LLMChain(llm=llm, prompt=prompt)
    stuff_chain = StuffDocumentsChain(
        llm_chain=llm_chain,
        document_prompt=document_prompt,
        document_variable_name=document_variable_name,
    )
    output = stuff_chain.invoke(content_docs)["output_text"]
    return output

```

We won't go into details on how to setup our summarization chain, but the gist is that we take our documents (scraped content from all the URLs passed in) and stuff them into an llm chain for summarization. The output would then be fed back into the agents to use. Notice we also use `gpt-4-turbo` here as content could fill up the context window. Feel free to experiment with different LangChain prompt and chains in summarizing content.

## Setup crewAI

The guys over at crewAI have done a fantastic job with the framework that makes setting up agents a breeze. We'll take a page from their [documentation](https://docs.crewai.com/how-to/Creating-a-Crew-and-kick-it-off/) in configuring our agents for a research use case. First, we need to figure out the make up of our "crew" members.

- Senior researcher: Our main research agent that will kick off the research and use tools like scrape and Google search to find web articles.
- Writer: Our writer that will write the article

Now, next we have to define the tasks for each agent to do.

- Research fundamentals task: This task will highlight the strengths and weaknesses of the company we want to research
- Research technicals task: In addition to fundamentals, we want to research the technicals of stock price. Obviously, this isn't needed but let's pretend we're a medium-long term trader who looks at technical analysis to know when is a good time to buy a stock.
- Write task: Finally, we define the actual task for writing the article. We're going to focus on recent news about the company, overall market outlook, and then the industry in which the company operates in.

## Setup our crews

```py
from crewai import Agent

researcher = Agent(
    role="Senior Stock Researcher",
    goal="Stock researcher for company or ticker: {company}",
    verbose=True,
    memory=True,
    backstory="Driven by researching the next upcoming company stock that would make a good purchase",
    tools=[search_tool, scrape_and_summarize],
    allow_delegation=True,
)

writer = Agent(
    role="Writer",
    goal="Blog writer for company stock {company}",
    verbose=True,
    memory=True,
    backstory=(
        "A writer for many popular business magazines and journals covering companies and business."
    ),
    tools=[search_tool, scrape_and_summarize],
    allow_delegation=False,
)
```

Pretty straightforward setup. We give each agent the ability to search and scrape content because each may want to conduct those intermediary tasks on their own. We allow the researcher to delegate tasks to the writer if they choose to. We also enable memory usage so that each agent retains information during and a across executions.

## Setup our tasks

```py
from crewai import Task

research_fundamentals_task = Task(
    description="Research stock for {company} based on the fundamentals of the company for 2024 and beyond."
    "Focus on identifying the strengths and weaknesses for the given company and provide reasons for why the stock is a good or bad buy. Scrape search results by passing in urls to the scrape_and_summarize tool."
    "Based on the scraped content, your final report should clearly articulate the key points,"
    "its market opportunities, and potential risks of buying the stock. ONLY use scraped content from our search results for the report.",
    expected_output="A comprehensive 4-6 paragraphs long report on company stock.",
    tools=[search_tool, scrape_and_summarize],
    agent=researcher,
)

research_technicals_task = Task(
    description="Research the technicals of stock chart for {company} for 2024 and beyond."
    "Focus on identifying the strengths and weaknesses of what the charts and price are saying for the given company and provide reasons for why the stock is a good or bad buy based on this perspective. Scrape search results by passing in urls to the scrape_and_summarize tool."
    "Based on the scraped content, your final report should clearly articulate the key points. ONLY use scraped content from our search results for the report.",
    expected_output="A comprehensive 4-6 paragraphs long report on company stock.",
    tools=[search_tool, scrape_and_summarize],
    agent=researcher,
)

write_task = Task(
    description=(
        "Compose an insightful article on {company}."
        "Focus on recent news about the company, fundamental analysis like strengths and weaknesses, it's overall market outlook and the industry in which the company operates. Also, write an overview of it's stock from a technical analysis perspective."
        "This article should be easy to understand, engaging, and positive. ONLY use scraped content from our search results for the report."
    ),
    expected_output="A 6-8 paragraph article on {company}, formatted as markdown.",
    tools=[search_tool, scrape_and_summarize],
    agent=writer,
    async_execution=False,
    output_file="COST-post.md",
    human_input=True,
)
```

You can see we add some instructions on how to use the scrape tool and to only use content that was scraped so that the agents do not make something up or rely on the LLM's internal knowledge. We set the expected output and filename for our article formatted as markdown. Human input is set to `True` in case we want to review the output and make changes to the final article.

Next, we'll finalize the crew and kick things off.

```py
from crewai import Process
from crewai import Crew
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_fundamentals_task, research_technicals_task, write_task],
    process=Process.sequential,
    memory=True,
    cache=True,
    max_rpm=100,
    share_crew=False,
    output_log_file="crewai_spider.log",
)
```

Here the settings are pretty self explanatory. The process will be sequential as researching and writing a blog post has a similar workflow. We'll set memory and cache to `True` for performance, especially during testing to save on LLM costs.

Then kickoff the crew!

```py
result = crew.kickoff(inputs={"company": "Costco Wholesale"})
print(result)
```

Here we define the inputs of our crew agents. For our stock research it will be the company name. Notice that the `company` key is used throughout our previous code examples.

Example output of an agent performing Google search for `Costco Wholesale` stock analysis:

```shell
> Entering new CrewAgentExecutor chain...
I need to gather recent and relevant information regarding Costco Wholesale's business fundamentals, focusing on its strengths, weaknesses, market opportunities, and potential risks related to its stock. I will start by searching for recent articles, analyst reports, and financial news related to Costco Wholesale's performance and projections for 2024 and beyond.

Action: Search the internet
Action Input: {"search_query": "Costco Wholesale stock analysis 2024"}


Search results: Title: Costco's Stock Is Expensive, But It Could Quickly Go Higher If This ...
Link: https://www.fool.com/investing/2024/04/04/costcos-stock-is-expensive-but-it-could-quickly-go/
Snippet: ... Price. $779.04. Price as of May 9, 2024, 4:00 p.m. ET. Is a price hike to Costco's membership coming soon? Costco Wholesale (COST 2.05%) is one ...
---
Title: Costco (COST) Q2 2024 earnings - CNBC
Link: https://www.cnbc.com/2024/03/07/costco-cost-q2-2024-earnings.html
Snippet: Costco on Thursday missed Wall Street's revenue expectations for its holiday quarter, despite reporting year-over-year sales growth.
---
Title: Costco Wholesale Corporation Reports Second Quarter and Year-to ...
Link: https://investor.costco.com/news/news-details/2024/Costco-Wholesale-Corporation-Reports-Second-Quarter-and-Year-to-Date-Operating-Results-for-Fiscal-2024-and-February-Sales-Results/default.aspx
Snippet: Costco Wholesale Corporation Reports Second Quarter and Year-to-Date Operating Results for Fiscal 2024 and February Sales Results ; ASSETS.
---
Title: Costco Wholesale (COST) Stock Forecast and Price Target 2024
Link: https://www.marketbeat.com/stocks/NASDAQ/COST/price-target/
Snippet: The average twelve-month price prediction for Costco Wholesale is $694.48 with a high price target of $870.00 and a low price target of $550.00. Learn more on ...
---
Title: Costco Profit Tops Expectations But Revenue Growth Disappoints
Link: https://www.investopedia.com/costco-q2-fy2024-earnings-8605941
Snippet: Costco reported better-than-expected earnings for the second quarter of fiscal 2024, but revenue growth was lower than analysts had anticipated.
---
Title: Costco Wholesale Stock Has 11% Upside, According to 1 Wall ...
Link: https://www.fool.com/investing/2024/04/04/costco-wholesale-stock-upside-wall-street-analyst/
Snippet: ... Price. $787.19. Price as of May 10, 2024, 4:00 p.m. ET. This analyst is growing cautious about the company's prospects in the near term. Costco ...
---
```

And an agent reviewing the search results and executing our scrape tool.

```shell
Thought:
The search results provided multiple sources with insights into Costco's financial performance and stock analysis for 2024. To better understand Costco Wholesale's business fundamentals, strengths, weaknesses, market opportunities, and potential risks for its stock, I will scrape and summarize content from the top relevant URLs.

Action: scrape_and_summarize
Action Input: {"urls": ["https://www.fool.com/investing/2024/04/04/costcos-stock-is-expensive-but-it-could-quickly-go/", "https://www.cnbc.com/2024/03/07/costco-cost-q2-2024-earnings.html", "https://investor.costco.com/news/news-details/2024/Costco-Wholesale-Corporation-Reports-Second-Quarter-and-Year-to-Date-Operating-Results-for-Fiscal-2024-and-February-Sales-Results/default.aspx", "https://www.marketbeat.com/stocks/NASDAQ/COST/price-target/", "https://www.investopedia.com/costco-q2-fy2024-earnings-8605941"]}
```

And finally we have the output for our article report of our stock analysis for `Costco Wholesale`:

```markdown
# Comprehensive Analysis of Costco Wholesale in 2024

Costco Wholesale continues to excel in the competitive retail market, showcasing strong
financial results and strategic growth initiatives as of 2024. The company reported a
significant 9.4% year-over-year increase in net sales reaching $23.48 billion in March,
with e-commerce experiencing an 18.4% surge, reflecting Costco's adeptness in integrating
digital solutions into its business model.

## Enhanced Market Position and Membership Growth

Costco's strategic positioning is reinforced by its extensive global presence,
with 876 warehouses worldwide. The company has been focusing on both enhancing
the in-store experience and expanding its digital footprint. Notably, membership
dynamics have been positively influenced by stricter card checking, which has
led to increased sign-ups and sustained membership loyalty,
a critical factor in Costco's recurring revenue model.

## Financial Stability and Shareholder Returns

Despite a slight shortfall in expected quarterly revenue, with $58.44 billion
reported against projections of $59.16 billion, Costco's financial health
remains robust, evidenced by a net income of $1.74 billion. The company’s
commitment to shareholder returns is evident from the recent increase in
its quarterly dividend from $1.02 to $1.16 per share, showcasing
its financial confidence and commitment to returning value to its investors.

## Navigating Challenges

Costco is not without its challenges; the company acknowledges potential impacts
from broader economic conditions and competitive pressures. The
ongoing geopolitical tensions, such as the Ukraine conflict, could pose
supply chain risks. However, Costco's diversified global operations
provide a buffer against localized economic disruptions.

## Detailed Technical Stock Analysis

Turning to technical analysis, Costco's stock has demonstrated a strong
upward trend, with a recent all-time high of $787.45. The stock's 52-week
range from $476.75 to $787.19 illustrates significant volatility but
also robust recovery and growth. Technical indicators suggest continued
bullish trends, although the Relative Strength Index (RSI) nearing 70
points to potential overbought conditions, hinting at possible short-term
pullbacks. Investors should look for support levels at around $750 as
potential buying points during dips.

## Industry Perspective

Within the Retail - Discount & Variety Industry, Costco maintains a competitive
advantage through its bulk-selling membership model, which is distinct from
competitors like TJX and Target. This model has consistently driven high
volume sales and customer retention through economic cycles, positioning
Costco favorably against industry peers.

## Forward-Looking Statements

Costco's forward-looking strategies include increasing its investment in
technology and infrastructure to further enhance its e-commerce capabilities
and improve operational efficiencies. The company's proactive approach in
adapting to consumer trends and technological advancements bodes well for
its sustained growth.

## Conclusion

In summary, Costco's performance in 2024 has been marked by impressive sales
growth, strategic market positioning, and strong financial health. While
mindful of market risks, the company's ongoing investments in digital
transformation and global expansion are likely to foster continued growth.
Investors and stakeholders should remain optimistic about Costco's market
trajectory while staying cautious of external economic and geopolitical
factors that could influence the retail sector.
```

See the [full code here](https://gist.github.com/gbertb/c04216260cfa70583462cbf2f3a0260b).

So now we got a full working code that demonstrates the power of using Spider as a scraping tool used by our agents. Thanks for following along! Stay tuned for more guides.
