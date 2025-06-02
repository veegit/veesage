+++
title = 'Agentic Platform - Chapter 3: Improve with Windsurf'
date = 2025-05-21T21:55:00-07:00
series = ["Agentic Platform"]
+++

[Previously using ChatGPT Plus](/articles/playing-with-chatgpt-coding) to build the frontend for the agent platform, I found it to be a bit slow with lot of back and forth. I then decided to try out Windsurf IDE.

While testing the Agent I encountered 3 problems
1. Agent was not able to remember the context e.g.
```
User: Who was Bill Clinton?
Agent: He was the 42nd President of USA
User: Where was he born?
Agent: Who are you referring to when you say "he"
```
2. Agent was not invoking skills properly like Whenever I ask to look for news and anything which requires a search, it doesn't. Even though the agent has access to web_search skill. E.g.
```
User: Any recent news about OpenAI and Jony Ive
Agent: Recently, there was an announcement that Jony Ive's design firm, LoveFrom, has partnered with Airbnb to work on designing futuristic homes. However, I couldn't find any recent news about Jony Ive being involved with OpenAI
```
3. Skills are not easy to configure and needed lot of hardcoding.

### One Solution to all problems - Invocation Pattern System
We built a configuration-driven system that lets skills declare their own activation patterns. Think of it like skills raising their hands saying "Pick me! Pick me!" when they see a query they can handle.

{{< mermaid >}}
graph TD
    A[User Query] --> B[Pattern Matcher]
    B --> C{Match Found?}
    C -->|Yes| D[Extract Parameters]
    C -->|No| E[Use Default Skill]
    D --> F[Execute Skill]
    E --> F
    F --> G[Return Response]
{{< /mermaid >}}

Windsurf was able to think and rethink and came up with a clever configuration based system

#### 1. Architecture
````Python
from pydantic import BaseModel, Field
from typing import List, Dict, Any, Optional

class InvocationPattern(BaseModel):
    """The secret sauce that makes skills smart!"""
    
    pattern: str = Field(..., description="What to look for in user queries")
    pattern_type: str = Field(default="keyword", description="How to match: 'keyword', 'regex', 'startswith', 'contains'")
    description: str = Field(..., description="Human-readable explanation")
    priority: int = Field(default=1, description="Higher = more important (1-10 scale)")
    sample_queries: List[str] = Field(default_factory=list, description="Example queries that trigger this pattern")
    parameter_extraction: Optional[Dict[str, Dict[str, Any]]] = Field(
        default=None, 
        description="The magic parameter extraction rules"
    )
````

#### 2. Usage
```Python
pattern = InvocationPattern(
    pattern="news about OpenAI and Jony Ive",
    pattern_type="keyword",
    description="Looks for news about OpenAI and Jony Ive",
    priority=5,
    sample_queries=["news about OpenAI and Jony Ive", "latest news about OpenAI and Jony Ive"],
    parameter_extraction={
        "company": {
            "type": "keyword",
            "key": "company"
        }
    }
)
```

#### 3. Parameter Extraction - The Smart Part
```Python
web_search_patterns = [
    InvocationPattern(
        pattern="latest|recent|news",
        pattern_type="keyword",
        description="Catches queries about current events",
        priority=5,
        sample_queries=[
            "What's the latest on quantum computing?",
            "Recent breakthroughs in AI",
            "Show me news about Mars exploration"
        ],
        parameter_extraction={
            "query": {
                "type": "content",  # Use entire user query
                "description": "The full search query"
            },
            .
            .
```

#### A Sample Skill Defintion

{{< githubscrollablefile user="veegit" repo="agent-platform" file="services/skill_service/skills/web_search.py" lang="python" height="500px" >}}

### Learnings
1. Windsurf is smart and can optimize code very well. It introduced a fast path to skip the reasoning path and directly invoke the web-skill. 
2. It tends to lose track of context and make things even more complicated. This is expected of any junior/mid level engineer. Like everyone else, its needs a break aka a new session :)
3. Langraph used by the Agent Service is a great way to structure the agent's behavior and make it more maintainable and uses a graph representation to structure reasoning, state, skills invocation and response formulation.
{{< mermaid >}}
graph TB
    subgraph "Reasoning Phase"
        D[Memory Manager]
        D --> E[Reasoning Node]
        E --> F{Match Invocation Patterns}

        subgraph "Pattern Matching"
            F --> |No Match| G[Direct Response]
            F --> |Match Found| H[Select Highest Priority Pattern]
            H --> I[Extract Parameters]
        end
    end

    subgraph "Skill Registry"
        SR[Skill Registry] --> |Provide Skills| F
        SR --> |Skill Definitions| SK1[Web Search Skill]
        SR --> |Skill Definitions| SK2[Summarize Text Skill]
        
        subgraph "Invocation Patterns" 
            SK1 --> |Contains| P1[Pattern: news, latest, etc.]
            SK1 --> |Parameter Extraction| PE1[Extract query, search_type]
            SK2 --> |Contains| P2[Pattern: summarize, condense, etc.]
            SK2 --> |Parameter Extraction| PE2[Extract text, format]
        end
    end

    subgraph "Skill Execution Phase"
        I --> J[Skill Service Client]
        J --> K[Skill Validator]
        K --> L[Skill Executor]
        L --> M[Execute Selected Skill]
        M --> N[Store Skill Result]
    end

    subgraph "Response Formulation Phase"
        N --> O[Response Formulation Node]
        G --> O
        O --> P[Generate Human-Friendly Response]
        P --> Q[Send Response to User]
    end

    %% Data flow connections
    Q --> |Update| D
    SR <--> |Redis Storage| DB[(Redis)]
    N --> |Store Results| DB
{{< /mermaid >}}
