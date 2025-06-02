+++
title = 'Agentic Platform - Chapter 1: Build with Claude Code'
date = 2025-05-18T20:36:05-07:00
series = ["Agentic Platform"]
+++

So, it was Sunday night, and after putting our toddler to sleep, I was finally able to sit down and experiment with Claude Code to build an Agentic Platform. I’ve been a long-time lurker on Hacker News and had heard great things about Claude Code, so I just had to try it. As a big Claude user, I signed up for Claude Code in the Developer Console, set up my credit card with $20 in credits, and was ready to go.

I was intrigued by the feasibility of building an agentic platform—a basic Chat UI where you can select an agent and start chatting. You can create your own agent by passing a prompt and a list of skills.

With a bit of help from Claude Chat, I was able to design a simple architecture for the platform.

{{< mermaid >}}
graph TD
    ClientApp[Client App]
    APILayer[Simple API Layer]
    AgentLifecycle[Agent Lifecycle Service]
    AgentService[Agent Service]
    SkillService[Skill Service]
    Redis[Redis Storage]

    ClientApp --> APILayer
    APILayer --> AgentLifecycle
    APILayer --> AgentService
    APILayer --> SkillService
    AgentLifecycle --> AgentService
    AgentService --> SkillService
    SkillService --> Redis
{{< /mermaid >}}

After setting up the Claude Code CLI on my local machine, I was able to prompt it to build a scaffolding of the platform while referencing the CLAUDE.md file in the repository. To my surprise—and at the cost of an additional $10 worth of prompting—it was able to generate a working API-based application, triggering the skills.

For web search and LLM functionalities, I did some cross-referencing and ultimately chose SerpAPI for web search and Groq for LLM and related skills and they have generous free tiers.

During prompting, I made sure to specify the use of LangGraph for the Agent Service. Furthermore, from an operational simplicity perspective, I asked it to Dockerize the entire service in one repository. This will be useful later.


:bulb: Tips to operate with Claude Code
1. Familiarize yourself with https://www.anthropic.com/engineering/claude-code-best-practices
2. Use the --verbose flag to get more details about the prompt
3. To reduce your token usage, ensure you don't ask it to generate tests or write comments or verbose documentation. This will save you some credits but at the cost of figuring out the code
4. Develop the system iteratively and prompt by prompt e.g 
    1. Initialize Your Project Structure
    2. Implement Shared Components
    3. Setup Redis Integration
    4. Implement the Skill Service
    5. Implement the Agent Lifecycle Service
    6. Implement the Agent Service
    7. Implement the API Service
    8. Create Main Application Entry Point
    9. Add a README
    10. Test the application


Here’s the CLAUDE.md file I used to generate the scaffolding.

{{< githubscrollablefile user="veegit" repo="agent-platform" file="CLAUDE.md" lang="markdown" height="500px" >}}
