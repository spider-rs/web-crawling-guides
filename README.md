<a href="https://spider.cloud" target="_blank">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="images/spider-logo-github-dark.png" style="max-width: 100%; width: 300px; margin-bottom: 20px">
    <img alt="Spider Logo" src="/images/spider-logo-github-light.png" width="300px">
  </picture>
</a>

# Spider Web Crawling and Scraping Guides

This repo contains a collection of guides on how to effectively use the Spider service to crawl or scrape. Contributors are welcome! 😁

## Collection

- [Using the Spider API](spider-api.md)
- [How to Use Proxy Mode](proxy-mode.md)
- [LangChain + Groq + Spider = 🚀 (Integration Guide)](langchain-groq.md)
- [CrewAI Spider Stock Research](crewai-spider-research-agent.md)
- [Extracting Contacts](extracting-contacts.md)
- [Automated Cold Email Outreach Using Spider](auto-email-response-outreach.md)
- [How to Archive Full Website](website-archiving.md)
- Building A Speedy Resilient Web Scraper for RAG AI ([Part 1](building-a-speedy-resilient-web-scraper-for-rag-ai-part1-preparing.md), [Part 2](building-a-speedy-resilient-web-scraper-for-rag-ai-part2-scaling-up.md))
- [Agents from Scratch](ai-agent-from-scratch.md)

## Anti-Bot Detection

Spider, combined with the [`headless-browser`](https://github.com/spider-rs/headless-browser) repo, achieves **full stealth** against leading bot detection services — even when running fully headless.

Our techniques make Spider the most powerful crawling stack available today, providing an invisible footprint while scraping at scale.

Below are some screenshots proving Spider's stealth against major bot detectors:

| Detector                                 | Screenshot                                                                                               |
| :--------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| BrowserScan.net Bot Detection            | ✅ [View Screenshot](images/anti_bot/www_browserscan_net_bot_detection.png)        |
| Bot Detector Rebrowser                   | ✅ [View Screenshot](images/anti_bot/bot_detector_rebrowser_net.png)                 |
| SammySoft Bot Ecom                       | ✅ [View Screenshot](images/anti_bot/bot_sannysoft_com.png)                         |
| Device and Browser Info (Are You a Bot?) | ✅ [View Screenshot](images/anti_bot/deviceandbrowserinfo_com_are_you_a_bot.png) |
| Fingerprint Ecom Playground              | ✅ [View Screenshot](images/anti_bot/demo_fingerprint_com_playground.png)            |
| Device and Browser Info - Device Test    | ✅ [View Screenshot](images/anti_bot/deviceandbrowserinfo_com_info_device.png)       |
| Creepjs - Device Test                    | ✅ [View Screenshot](images/anti_bot/abrahamjuliot_github_io_creepjs.png)         |

Spider is designed for **extreme evasion**, **high concurrency**, and **human-like behavior**, allowing you to dominate even the most protected websites.

## Contribute

We're happy to accept requests in the issue tracker, improvements to the content, and additional guides.
