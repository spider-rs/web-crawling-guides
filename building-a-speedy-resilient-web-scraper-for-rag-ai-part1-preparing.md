# Building A Speedy Resilient Web Scraper for RAG AI: Part 1 — Preparing

> “Building a RAG AI is easy; building a RAG AI at scale is incredibly difficult.” — Jerry Liu of LlamaIndex.

As part of my ongoing series on how to build a scalable RAG AI, today I’ll be diving deep into web scraping, the automated extracting of data from web pages. Many RAG AIs need at least a little bit of data gathered from various sites around the web. Typical examples include gathering data for customer support chatbots, generating personalized recommendations, and extracting information for market analysis.

For my project, [The Journey Sage Finder](https://lowryonleadership.com/2024/04/02/the-journey-sage-finder/), I needed to scrape [thejourneysage.com](https://thejourneysage.com/) for answers to common questions. This required only a single page worth of data. [My Virtual College Advisor](https://lowryonleadership.com/2024/05/27/inside-the-virtual-college-advisor-a-deep-dive-into-rag-ai-and-agent-technology/) required a much more extensive scrape of over 6,200 college and university websites comprising well over a million unique web pages.

What worked fine for the one page The Journey Sage failed spectacularly when trying to bring it to scale. In the hopes of saving the next person some time, I’m sharing much of what I encountered on this journey.

## Upfront Preparation

Any large-scale scraping requires some upfront preparation. Upfront, you should think about:

1. What data you need.
2. What tools to use.
3. Testing on a single website first.
4. Scale Up! Have a plan for problems that may arise.
5. Error First Thinking. Expect some cleanup.

In this post, we will cover the first three items: understanding what data you need, choosing your tools, and testing on a single website. The next post will cover scaling up your operations and adopting an error-first thinking approach.

## Understand What Data You Need

When gathering data for a RAG AI, less is often more. Extraneous data erodes the quality of the semantic searches and thus the answers generated. Each extra bit of data stored adds to storage costs and overhead for embedding and retrieval. Good quality data in as small a package as you can get, with no extraneous data, is what you are shooting for.

When scraping, if there’s one piece of advice I’d stress above all others, it’s that **keeping the data to just what you really need** is vital.

A typical website has huge amounts of data used for styling, headers, footers, and structure. For instance, if you look at the HTML of the homepage for my [website](https://www.lowryonleadership.com), you see 934 lines of HTML, but really only about 16 lines you’d want to use in a RAG. That’s 2% of the page that has useful data for your RAG AI, while 98% is formatting, navigation, and the like. Think hard about what data you truly need.

## Choose Your Tools

You will need some way to handle:

### Scraping the websites
   
I tried a number of different options before settling on [Spider](https://spider.cloud/). I started just calling web pages directly. After all, I reasoned, all a browser really does is make a call and get the HTML back. I should be able to do so as well. This worked fine for a simple WordPress site, but fell apart quickly at scale. Calls tended to be rejected by web servers and I got a number of certificate errors. I then tried Chromium, Google’s web browser foundation. This worked but had many intricacies, a steep learning curve, numerous errors, and was a bit slow.

Spider does require a small fee, but is super quick, easy to use, and has good customer support. Importantly, it runs headless, meaning that my system doesn’t open up a browser for each web page accessed. It also fits seamlessly with Llama Index, which I use extensively.

### Benefits of Spider:

- Super fast.
- API easy to use.
- Support for streaming and asynchronous processing.
- Headless.
- Low memory.

I will talk more about these benefits when I discuss scaling up in the next post.

## Cleaning the Returned Data

For cleaning the HTML, I chose [Beautiful Soup](https://beautiful-soup-4.readthedocs.io/en/latest/) for its great documentation and reputation. It effectively stripped out unnecessary elements, leaving only the 2% of data needed for my RAG AI.

Originally, I had saved the headers, footers, and menus. My thinking was that this contained valuable information. **What it really contained was a lot of noise**. For instance, let’s say I’m scraping school websites and the side menu lists all of the academic departments. The problem is that a semantic search on “Economics” gets a hit on dozens or hundreds of pages that actually are about other departments. While those pages will likely get weeded out by the reranker, **it’s still unnecessary overhead which can only erode accuracy and performance**.

I ended up culling pretty much everything from the page except the headers and the main text. To be specific, I removed everything with the following tags: ‘script’, ‘style’, ‘nav’, ‘footer’, ‘header’, ‘aside’, ‘form’ as well as the following classes: ‘menu’, ‘sidebar’, ‘ad-section’, ‘navbar’, ‘modal’, ‘footer’, ‘masthead’, ‘comment’, ‘widget’.

Whereas I tried half a dozen different scrapers before settling on Spider, the only thing I tried before using Beautiful Soup was some custom-built code to strip styling. Beautiful Soup has produced high-quality output for me and is quick enough for my purposes, but I can’t attest to how well it works compared to any similar utility.

## Testing on a Single Website

Before scaling up, it’s crucial to test your scraper on a single website. This allows you to identify and resolve issues without dealing with the complexity of multiple sites. Here’s how you can do it:

1. **Select a Representative Site:** Choose a website that is representative of the type of data and structure you expect to encounter in your larger scrape.
2. **Run Your Scraper:** Execute your scraper on this site and carefully analyze the data you retrieve.
3. **Adjust Your Process:** Make any necessary adjustments to your scraping and cleaning process based on the results.
4. **Validate Your Data:** Ensure that the data you have gathered is accurate, relevant, and clean.

This step helps you fully understand the data before ramping up to multiple websites, reducing the risk of encountering major issues later.

## Storing the Returned Data

In this example, I’m using MongoDB. For smaller implementations, I actually use local flat files with a SQLite database for coordination. I’m not sure this choice matters too much in this phase. It can have enormous impacts on performance in the vector search phase, but that’s another post…

Instead of just taking the cleaned data and storing it, I suggest adding a few additional steps:

### 1. Hashing the content to check for duplicates.

To my surprise, many sites have a large amount of duplicate content. Sometimes this is different URLs pointing to the same page, other times it’s basic information repeated. Depending on the website, I was finding anywhere between 0% duplicates and 70% (!!!) duplicates. At first, I had a process go through and clean out duplicates, but the issue was so pervasive I decided that I needed to stop it at the source instead of after the fact. I added a hash to each website saved. If the hash already existed, I didn’t save the duplicate page.

To reiterate, keeping just the data you need is vital to accuracy and performance. Duplicates in the database mean that when your semantic search retrieves the top 5 websites, it might retrieve 5 exact copies.

Note: Don’t include the URL in the data that is hashed or multiple URLs with the same content won’t be detected as duplicates.

### 2. Make sure to add basic sanity checks on each page.

In my scraping, about 4% of the returned web pages had very little data once cleaned. Usually, this is because the web page had a header and a picture on it.

This **4% caused 90% of my troubles during semantic searches**. For example, a page that said “Economics Department” and had a picture of the department staff would score very high on a semantic search for “Economics.” Ironically, it would score better than a page that described the Economics Department in detail because the extra detail, while important to anyone querying the topic, made the semantic value of the page differ from the base term itself.

Once those 4% were removed, average [evaluation scores](https://lowryonleadership.com/2024/05/30/evaluating-vector-search-performance-on-a-rag-ai-a-detailed-look/) went from 4.6 to 9.5 (scale of 1–10. Top 3 looked at for relevance to query). That’s a huge change by just getting rid of a small amount of data.

### 3. Size Matters.
Another major issue was large pages. These tended to be large PDFs. Sometimes these would be 100MB or more. These PDFs are not valuable for my use case. I automatically ignore any document over 1MB. Ultimately, for this project, I decided to also ignore everything in the cleaned data beyond the first 7,000 tokens. Why?

At first, I was doing the [standard chunking](https://medium.com/@anuragmishra_27746/five-levels-of-chunking-strategies-in-rag-notes-from-gregs-video-7b735895694d) of larger pages into smaller segments. Depending on your situation, this may or may not be wise:

a. In [The Journey Sage Finder](https://lowryonleadership.com/2024/04/02/the-journey-sage-finder/), which sometimes dealt with long videos that handled multiple different topics, this sort of chunking was important to retain information of the topics discussed in the video. It improved semantic search results as each chunk had its own embedding and was thus searched separately.

b. For [My Virtual College Advisor](https://lowryonleadership.com/2024/05/27/inside-the-virtual-college-advisor-a-deep-dive-into-rag-ai-and-agent-technology/), this chunking was not helpful. Tests using the evaluator showed slightly lower evaluation scores when chunking. This is because school web pages tend to be single subject, as opposed to long videos that might cover several subjects. The first 7,000 tokens have an extremely high likelihood of capturing the semantic meaning of the entire page. Indeed, the longer the page, the more likely it will veer off topic enough to make the page less relevant. So I decided I wouldn’t bother to process, chunk and save this extra data since it doesn’t improve query accuracy.

## Conclusion

In this first part, we have laid the groundwork for building a robust web scraper for your RAG AI. By understanding your data needs, choosing the right tools, and testing on a single website, you can ensure a smooth and efficient scraping process. In the next part, we delve into scaling up your scraping operations, tackling issues of performance, and maintaining resilience. Read [Part 2](building-a-speedy-resilient-web-scraper-for-rag-ai-part2-scaling-up.md), where we discuss scaling up.

- Author: Troy Lowry
- Twitter: [@Troyusrex](https://x.com/Troyusrex)
- Read more: [https://lowryonleadership.com](https://lowryonleadership.com)