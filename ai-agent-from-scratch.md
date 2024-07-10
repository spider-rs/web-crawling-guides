# Guide - Build an AI Agent from Scratch

AI agents are revolutionizing how we process and interact with information. By combining language models with web search capabilities, we can create assistants that not only understand our queries but can actively research and provide comprehensive answers. This guide will show you how to harness this power.

## Setup

First, let's set up our environment and install the necessary dependencies.

### Install Required Packages

Install the required packages using pip:

```bash
pip install python-dotenv openai spider-client colorama
```

- `python-dotenv`: Manages environment variables
- `openai`: Interfaces with OpenAI's powerful language models
- `spider-client`: Scraping, crawling and web searching (all of [Spiders](https://spider.cloud/) capabilities)
- `colorama`: Adds color to our console output for better readability

### Environment Variables

Create a `.env` file in your project root and add your API keys:

```bash
OPENAI_API_KEY=<your_openai_api_key_here>
SPIDER_API_KEY=<your_spider_api_key_here>
```

## Building the AI Research Agent

Let's break down the process of building our AI agent into steps.

### Step 1: Import Dependencies and Set Up

```python
import os
from dotenv import load_dotenv
import openai
from spider import Spider
from typing import List, Dict, Any
from colorama import init, Fore


init(autoreset=True)
load_dotenv()

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
SPIDER_API_KEY = os.getenv("SPIDER_API_KEY")
```

This section sets the stage for our agent. We're importing necessary libraries and loading our environment variables. The use of `colorama` will make our console output more visually appealing and easier to read.

### Step 2: Create the AIResearchAgent Class

The `AIResearchAgent` class is the core of our AI assistant. It encapsulates all the functionality we'll be building, providing a clean and organized structure for our code.

```python
class AIResearchAgent:
    def __init__(self, openai_api_key: str, spider_api_key: str):
        self.openai_client = openai.OpenAI(api_key=openai_api_key)
        self.spider_client = Spider(spider_api_key)
```

This initializer sets up our connections to the OpenAI and Spider APIs, preparing our agent for action.

### Step 3: Implement Web Search Functionality

Web search is a crucial capability of our agent. By leveraging Spider's API, we can fetch relevant information from across the internet, providing our agent with up-to-date data to work with. And thanks to spider's speed, we don't have to wait ages for this data to be returned.

```python
def search(self, query: str, limit: int = 5) -> List[Dict[str, Any]]:
    """Perform a web search using Spider."""
    params = {"limit": limit, "fetch_page_content": False}
    print(f"{Fore.GREEN}Searching for: {query}")
    results = self.spider_client.search(query, params)
    return results
```

This method allows our agent to cast a wide net across the web, gathering diverse information to inform its responses.

### Step 4: Implement OpenAI Request Helper

```python
def openai_request(self, system_content: str, user_content: str) -> str:
    """Helper method to make OpenAI API requests."""
    response = self.openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system_content},
            {"role": "user", "content": user_content}
        ]
    )
    return response.choices[0].message.content
```

This helper method streamlines our interactions with OpenAI's API, abstracting the complexities of API calls and allowing us to focus on the core functionality of our agent.

### Step 5: Implement Text Summarization (this method is not used in the code below, but you can easily implement it by calling it before the `combined_summary` variable defined in the research method)

Summarization is a powerful feature that allows our agent to distill large amounts of information into concise, digestible chunks. This is particularly useful when dealing with lengthy web content. We don't use this function, but it is here for you as a little "task", if you want to implement it to our agent.

```python
def summarize(self, text: str) -> str:
    """Summarize the given text using OpenAI."""
    print(f"{Fore.BLUE}Summarizing...", text)
    return self.openai_request(
        "You are a helpful assistant that summarizes text.",
        f"Summarize this text in 2-3 sentences: {text}"
    )
```

This method uses OpenAI to summarize text.

### Step 6: Implement Answer Evaluation

```python
def evaluate(self, question: str, summary: str) -> str:
    """Evaluate if the summary answers the question."""
    print(f"{Fore.MAGENTA}Evaluating...")
    evaluation = self.openai_request(
        "You are an AI research assistant. Your task is to evaluate if the given summary answers the user's question.",
        f"Question: {question}\n\nSummary:\n{summary}\n\nDoes this summary answer the question? If it does, write exactly: 'does answer the question'. If not, explain why."
    )
    print(f"{Fore.MAGENTA}Evaluation: {evaluation}")
    return evaluation
```

This method adds a layer of intelligence to our agent. By evaluating whether a summary answers the original question, our agent can determine if it needs to continue searching or if it has found a satisfactory answer. This is the core and what makes this a [level 3](https://arxiv.org/pdf/2405.06643#:~:text=Inspired%20by%20the%206%20levels,%2FRL%2Dbased%20AI%2C%20with) agent.  

### Step 7: Implement Search Query Formation

Forming effective search queries is an art, and the users query might not always be formed as a search query:
- User query: What is the wheater is Boston?
- Search query: Boston weather
This method leverages OpenAI's language understanding to create queries that are more likely to yield relevant results.

```python
def form_search_query(self, user_query: str) -> str:
    """Form a search query from the user's input."""
    search_query = self.openai_request(
        "You are an AI research assistant. Your task is to form an effective search query based on the user's question.",
        f"User's question: {user_query}\n\nPlease provide a concise and effective search query to find relevant information."
    )
    return search_query
```

By refining user queries, our agent can perform more targeted and efficient web searches.

### Step 8: Implement Final Answer Formation

This is where our agent truly shines. By using the information it has gathered (and evaluated to be sufficient in asnwering the user query), it can form comprehensive answers to complex questions.

```python
def form_final_answer(self, user_query: str, summary: str) -> str:
    """Form a final answer based on the user's query and the summary."""
    final_answer = self.openai_request(
        "You are an AI research assistant. Your task is to form a comprehensive answer to the user's question based on the provided summary.",
        f"User's question: {user_query}\n\nSummary of research:\n{summary}\n\nPlease provide a comprehensive answer to the user's question based on this information."
    )
    print(f"{Fore.GREEN}Formed final answer.")
    return final_answer
```

This method demonstrates the agent's ability to understand context, synthesize information, and communicate clearly.

### Step 9: Implement Question Refinement

```python
def refine_question(self, original_question: str, evaluation: str) -> str:
    """Refine the search question based on the evaluation."""
    print(f"{Fore.CYAN}Refining...")
    return self.openai_request(
        "You are an AI research assistant. Your task is to refine a search query based on the original question and the evaluation of previous search results.",
        f"Original question: {original_question}\n\nEvaluation of previous results: {evaluation}\n\nPlease provide a refined search query to find more relevant information."
    )
```

The ability to refine questions based on previous results is what makes our agent truly adaptive. This iterative approach allows the agent to hone in on the most relevant information, improving its research capabilities with each iteration.

### Step 10: Implement the Main Research Loop

Now we come to the heart of our AI agent - the main research loop. This is where all the pieces come together to create a powerful, autonomous research assistant.

```python
def research(self, user_query: str, max_iterations: int = 5) -> str:
    """Perform research on the given question."""
    print(f"{Fore.BLUE}Starting research for: {user_query}")
    
    for iteration in range(max_iterations):
        print(f"{Fore.YELLOW}Iteration {iteration + 1}/{max_iterations}")

        search_query = self.form_search_query(user_query)
        search_results = self.search(search_query)
        # OPTIONAL: call the summarize method here to summarize the search results
        combined_summary = "\n".join([result['description'] for result in search_results['content']])
        evaluation = self.evaluate(user_query, combined_summary)

        if "does answer the question" in evaluation.lower():
            final_answer = self.form_final_answer(user_query, combined_summary)
            return f"{Fore.GREEN}Final Answer:\n{final_answer}\n\nBased on:\n{combined_summary}"

        user_query = self.refine_question(user_query, evaluation)
        
    return f"{Fore.RED}Couldn't find a satisfactory answer after {max_iterations} iterations. Last summary:\n{combined_summary}"
```

This method orchestrates the entire research process, from forming initial queries to delivering final answers. It showcases the agent's ability to:

- Form effective search queries
- Evaluate the relevance of search results
- Refine and give feedback on its approach based on intermediate results
- Synthesize information into a coherent final answer

### Step 11: Implement the Main Function

Finally, let's create an interactive interface for users to engage with our AI research agent:

```python
def main():
    agent = AIResearchAgent(OPENAI_API_KEY, SPIDER_API_KEY)
    while True:
        user_input = input("What would you like to research? (Type 'exit' to quit): ")
        if user_input.lower() == 'exit':
            break
        result = agent.research(user_input)
        print(result)

if __name__ == "__main__":
    main()
```

This main function brings everything together, allowing users to interact directly with the AI agent and experience its research capabilities firsthand.

## Conclusion

You now have a fully functional AI research agent that can:

- Form web searches
- Evaluate if the search results are sufficient
- Give feedback to itself, to improve the search query if the serch results were insufficient
- Form final answer based on the search results gathered

## Complete Code

You can find the complete code for this guide down below:

```python
import os
from dotenv import load_dotenv
import openai
from spider import Spider
from typing import List, Dict, Any
from colorama import init, Fore


init(autoreset=True)
load_dotenv()

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
SPIDER_API_KEY = os.getenv("SPIDER_API_KEY")

class AIResearchAgent:
    def __init__(self, openai_api_key: str, spider_api_key: str):
        self.openai_client = openai.OpenAI(api_key=openai_api_key)
        self.spider_client = Spider(spider_api_key)

    def search(self, query: str, limit: int = 5) -> List[Dict[str, Any]]:
        """Perform a web search using Spider."""
        params = {"limit": limit, "fetch_page_content": False}
        print(f"{Fore.GREEN}Searching for: {query}")
        results = self.spider_client.search(query, params)
        return results

    def _openai_request(self, system_content: str, user_content: str) -> str:
        """Helper method to make OpenAI API requests."""
        response = self.openai_client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": system_content},
                {"role": "user", "content": user_content}
            ]
        )
        return response.choices[0].message.content

    def summarize(self, text: str) -> str:
        """Summarize the given text using OpenAI."""
        print(f"{Fore.BLUE}Summarizing...")
        return self._openai_request(
            "You are a helpful assistant that summarizes text.",
            f"Summarize this text in 2-3 sentences: {text}"
        )

    def evaluate(self, question: str, summary: str) -> str:
        """Evaluate if the summary answers the question."""
        print(f"{Fore.MAGENTA}Evaluating...")
        evaluation = self._openai_request(
            "You are an AI research assistant. Your task is to evaluate if the given summary answers the user's question.",
            f"Question: {question}\n\nSummary:\n{summary}\n\nDoes this summary answer the question? If it does, write exactly: 'does answer the question'. If not, explain why."
        )
        return evaluation

    def form_search_query(self, user_query: str) -> str:
        """Form a search query from the user's input."""
        search_query = self._openai_request(
            "You are an AI research assistant. Your task is to form an effective search query based on the user's question.",
            f"User's question: {user_query}\n\nPlease provide a concise and effective search query to find relevant information."
        )
        return search_query

    def form_final_answer(self, user_query: str, summary: str) -> str:
        """Form a final answer based on the user's query and the summary."""
        final_answer = self._openai_request(
            "You are an AI research assistant. Your task is to form a comprehensive answer to the user's question based on the provided summary.",
            f"User's question: {user_query}\n\nSummary of research:\n{summary}\n\nPlease provide a comprehensive answer to the user's question based on this information."
        )
        print(f"{Fore.GREEN}Formed final answer.")
        return final_answer

    def refine_question(self, original_question: str, evaluation: str) -> str:
        """Refine the search question based on the evaluation."""
        print(f"{Fore.CYAN}Refining...")
        return self._openai_request(
            "You are an AI research assistant. Your task is to refine a search query based on the original question and the evaluation of previous search results.",
            f"Original question: {original_question}\n\nEvaluation of previous results: {evaluation}\n\nPlease provide a refined search query to find more relevant information."
        )

    def research(self, user_query: str, max_iterations: int = 5) -> str:
        """Perform research on the given question."""
        print(f"{Fore.BLUE}Starting research for: {user_query}")
        
        for iteration in range(max_iterations):
            print(f"{Fore.YELLOW}Iteration {iteration + 1}/{max_iterations}")
            
            search_query = self.form_search_query(user_query)
            search_results = self.search(search_query)
            # OPTIONAL: call the summarize method here to summarize the search results
            combined_summary = "\n".join([result['description'] for result in search_results['content']])
            evaluation = self.evaluate(user_query, combined_summary)
            
            if "does answer the question" in evaluation.lower():
                final_answer = self.form_final_answer(user_query, combined_summary)
                return f"{Fore.GREEN}Final Answer:\n{final_answer}\n\nBased on:\n{combined_summary}"
            
            user_query = self.refine_question(user_query, evaluation)
        
        return f"{Fore.RED}Couldn't find a satisfactory answer after {max_iterations} iterations. Last summary:\n{combined_summary}"

def main():
    agent = AIResearchAgent(OPENAI_API_KEY, SPIDER_API_KEY)

    while True:
        user_input = input("What would you like to research? (Type 'exit' to quit): ")
        if user_input.lower() == 'exit':
            break

        result = agent.research(user_input)
        print(result)

if __name__ == "__main__":
    main()
```

If you liked this guide, consider checking out Spider on Twitter and follow me (the author):
- **Spider Twitter:** [spider_rust](https://x.com/spider_rust)
- **William Espegren Twitter:** [@WilliamEspegren](https://x.com/WilliamEspegren)
