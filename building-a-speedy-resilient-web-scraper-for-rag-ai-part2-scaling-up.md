# Building a Fast and Resilient Web Scraper for Your RAG AI: Part 2 — Scaling Up

In the first part of this [series](building-a-speedy-resilient-web-scraper-for-rag-ai-part1-preparing.md), we covered understanding what data you need, choosing your tools, and testing your scraper on a single website. Now, we will delve into the challenges of scaling up your web scraping operations and adopting an error-first thinking approach. We also discuss in depth the importance of limiting the blast radius of any errors.

You can find the complete code for this project on [GitHub](https://github.com/Troyusrex2/RAG-AI-Scaling).

## Scale Up! Have a Plan For Problems

You WILL encounter problems. Scraping one web page is easy; scraping a million is anything but. Websites vary tremendously and what worked on the first 100 websites might fail on the 101st. It might also work on one website for 100 days and fail on the 101st. This is especially true for websites that use technologies like React or Drupal to supplement their HTML. Even with a great tool like [Spider](https://spider.cloud/), errors are a fact of scraping.

A few things to keep in mind:

1. **Errors:** Every web scraping program I tried gave me errors occasionally; Spider is no different. What is different is that Spider’s customer support helped me handle most issues quickly. Outright errors, where an exception was thrown, were okay because at least the problem was known and could be handled by error trapping.

2. **Stopping:** Sometimes the API I was calling simply failed to return any data.

3. **Few or no pages returned:** Having some pages on a site return with a good amount of data and others not at all is especially problematic for a RAG AI where incomplete data will make for incomplete results. Scraping sites made up of many sub-sites, which is especially prevalent at organizations such as universities that are made up of many quasi-independent sub-entities, often using different web technologies, is especially prone to this. To handle this, any website that returns fewer than 500 pages is flagged for review.

4. **Pages returned, but with little or no data:** While sometimes an error, this is more often a picture-heavy site that has little text with the pictures. Because of this, I only consider pages with at least 75 words of text worth saving. Depending on what your RAG AI is doing, your cutoff may differ.
   
5. **Spinning forever:** Like the ‘stopping’ above, this is where the system waits for the API to return data but for whatever reason, the data never returns. This is made worse by the fact that processing web pages is a highly variable process, depending not just on the complexity of the web page being scraped, but the latency of the internet and my machines. With other scraping APIs, some pages would take minutes. Spider is quick enough that if nothing is returned after 2 minutes it’s fine to assume a problem and move on.

6. **Out of memory:** Scraping can be a memory-intensive exercise. My previous scraper, using Chromium directly, would run out of memory on my 32GB i9 machine. In contrast, **I run Spider on a series of AWS t2.nanos, which have 0.5GB of memory and 1 CPU**. This is half the memory of a $35 Raspberry Pi (!!!). It runs out of memory approximately once every 300 websites (averaging 500 pages per website). When running a t2.small, with 2 GB of memory, I’ve never run out of memory. So I simply have it so that if a site runs out of memory, the website is later picked up by a t2.small and reprocessed. The t2.nano is about ¼ the price of the t2.small (much less when free tier services are factored in).
   
## Error First Thinking. Expect Some Cleanup

Since errors are inevitable, the important thing is to know when they happen and [handle them gracefully](https://lowryonleadership.com/2024/04/05/dannys-law-resilience-is-built-by-facing-adversity-not-by-avoiding-it/).

To ease handling, I assume each webpage is an error until proven otherwise. This dramatically improves my processing as it encouraged me to built in the following features:

1. **Error logging/Trapping:** All good systems include error logging and trapping. I built this in from the beginning. In this case, it was only the start of a larger fault-tolerant architecture.
   
2. **On Error, flag it and continue to the next:** I’m a big fan of Toyota’s [stop-the-line](https://businessmap.io/blog/stop-the-line) processing where when errors happen, the entire processing line stops and errors get resolved before any other processing is allowed to occur. That said, **I’m a bigger fan of sleep**. Stopping all processing when an issue occurs meant I woke up many mornings to completely stopped processing, often having lost 40 threads worth of processing for many hours.
   
Back before the speed of Spider, I projected I would need 4 months of processing time for 40 concurrent threads to complete my scraping. Losing 6 or 8 hours meant delaying launch by another day. This prompted me to just flag any errors and move on, allowing processing to continue. This did backfire a few times when an error would pop up that would propagate through the threads, causing lots of lost work and cleanup.

1. **Have a separate process to track and handle errors:** Delaying the review of errors until the morning didn’t mean ignoring them, far from it. It did mean I had to have time set aside to identify and work through all the errors from the previous night. Any errors not resolved, and the root cause mitigated, will just pop up again and again until it is resolved, so it’s vitally important to have these looked at.
   
2. **Be careful about concurrency: Be careful about concurrency:** When multiple processes or threads are running, ensure that any website is only worked on by one process. I used a simple processing flag in MongoDB to handle this.
   
3. **Limit Blast Radius, so when problems happen they are isolated:** This one is so important, it deserves its own section:

## Separate Processing of Requests — Limited Blast Radius

Originally, I had a nice multi-threaded scraper running on my beefy server. Again and again, I was hitting unexpected issues that would bring the entire system down. Be it a memory issue or a threading problem, issues I expected would be trapped and isolated instead impacted the entire system, meaning that an error on one thread would compromise the entire system. This meant that my beefy server that could scrape 40 websites at a time was actually a liability instead of an asset. Having architected many production systems, there are ways to architect around this using clusters, microservices, and the like, but this is a background process run infrequently and an outage doesn’t have the impact of a customer-facing system facing a failure. I needed a simpler, cheaper solution.

As previously mentioned, one problem with most of the other scrapers I tried was that they would bring up a browser to display every webpage being scraped. Being able to scrape a site without bringing up a browser is called running “headless.” I certainly tried running Chromium headless and had some success, but I encountered many new additional errors attempting to run headless. In particular, sites seemed more able to detect that I was scraping and would prevent me from accessing the site.

Running with the browser appearing, at first, was a benefit as I could see firsthand what the system was doing. It became a liability because all of those browser windows popping up on my server made doing anything else with the server difficult as it would occasionally grab focus of the system. There’s nothing quite like typing code and then being thrust onto a website mid-sentence. Playing a game on my computer while running the scraping was completely out of the question.

A bigger issue with having the browser pop up was that I could not use **low-cost cloud computing** to do this work as the lowest-cost cloud services don’t have a user interface. If I could find a headless, low-memory option, I could use these cloud services and then I could just “throw servers” at the problem. But without a headless option, that would be very expensive. Worse, managing a CLI-only t2.nano is simplicity. Managing a Windows GUI server is far more complex. Spider being able to run well on these machines was a game-changer.

As I mentioned above, these tiny servers would occasionally hit errors. Unfortunately, scraping websites will never be a “fire and forget” exercise. It will take constant oversight as technology changes to make sure everything keeps running correctly and quick intervention when it doesn’t. The internet and targeted websites are just changing so much that constant vigilance is required.

The fact that Spider ran headless with much lower memory requirements meant I could spin up 15 low-cost t2.nanos and have each run the scraper single-threaded. These nanos are actually on AWS’ free tier. My total cloud server cost to handle 1.2 million web pages was less than the cost of a cappuccino at Starbucks! My Spider costs did run several hundred dollars, but far less than the $1,000 in AWS costs I had originally planned and would have needed had I continued with the Chromium route.

## Conclusion

Building a resilient, fault-tolerant web scraper is crucial for the success of a scalable RAG AI. The journey from scraping a few pages to handling millions is fraught with challenges, but with the right tools and strategies, it becomes manageable. The key is to plan, choose your tools wisely, and be prepared for the inevitable issues that will arise.

Having my “spider legion” of 15 AWS t2.nano servers, backed by a single AWS t2.small server for the very rare high-memory website, I was able to complete in a week what I had expected would take four months (if all went well!). Spider’s low memory overhead combined with it being headless meant I could run it massively in parallel. This setup allowed for efficient, cost-effective scraping at scale.

## Key Takeaways:

- **Understand Your Data Needs:** Always aim to collect only the essential data to maintain high-quality semantic searches.
- **Choose Reliable Tools:** Tools like Spider for scraping can significantly streamline your process.
- **Plan for Errors:** Implement robust error logging and handling mechanisms to ensure your scraping process is resilient.
- **Optimize Storage:** Use strategies like hashing to eliminate duplicates and store only the necessary data.
- **Parallel Processing:** A tool like Spider allows you to use cloud services to run multiple scraping instances in parallel, which can drastically reduce the time required for large-scale scraping projects.
  
By leveraging these strategies, you can build a scalable, efficient, and resilient web scraping system that forms a robust foundation for your RAG AI. As the landscape of web technologies continues to evolve, staying adaptive and prepared for new challenges will be essential for ongoing success.

You can find the complete code for this project on Troy's [GitHub](https://github.com/Troyusrex2/RAG-AI-Scaling).

- Author: Troy Lowry
- Twitter: [@Troyusrex](https://x.com/Troyusrex)
- Read more: [https://lowryonleadership.com](https://lowryonleadership.com)