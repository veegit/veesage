+++
title = 'Agentic Platform - Chapter 5: Build Mult-Agent PoC with Codex'
date = 2025-06-28T00:06:00-07:00
series = ["Agentic Platform"]
ShowToc = false
TocOpen = false
+++

After the deploying the single agent platform framework to [Azure](/articles/playing-with-azure/), I wanted to make it multi-agent. The timing was perfect too, ChatGPT Codex was available "freely" to plus users and I decided to give it a spin. Eventually the codebase goes from “one hero handles it all” to “call in the specialists.” It’s like upgrading from the Swiss Army knife in your drawer to a whole toolbox—because even bots shouldn’t have to do everything themselves. 

Also another timely coincidence was the introduction of 3 seminal blogs around multi-agents. After going through it multiple times, I decided to chose Multi Agents with a Supervisor pattern. I would highly recommend these articles for building your knowledge as of <this blog post>.
1. Cognition - [Don't build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents)
2. LangChain - [Benchmarking Multi-Agent Architectures](https://blog.langchain.com/benchmarking-multi-agent-architectures/)
3. Anthropic - [How we built our multi-agent research system](https://www.anthropic.com/engineering/built-multi-agent-research-system)

### Why Add More Agents?
Originally, our agent could do web searches, summarize text, and ask follow-up questions. But what if a user wants the latest stock price? We created a shiny new finance skill for that. No more pretending we know Apple’s price off the top of our silicon heads.


{{< mermaid >}}
graph TD
    User --> Supervisor
    Supervisor --> ReasoningLLM
    Supervisor --> DelegationStore
    Supervisor --> DomainAgents
    Supervisor --> GeneralAgent
    DomainAgents --> SkillService
    GeneralAgent --> SkillService
    Supervisor --> User

    %% Grouping Domain Agents
    subgraph DomainAgents [Domain Agents]
        style DomainAgents fill:#d9fdd3,stroke:#999,stroke-width:1px
        ResearchAgent
        FinanceAgent
    end

    %% Explicit links from Supervisor to individual Domain Agents (needed for rendering)
    Supervisor --> ResearchAgent
    Supervisor --> FinanceAgent

    %% Coloring GeneralAgent
    style GeneralAgent fill:#d9fdd3,stroke:#999,stroke-width:1px
{{< /mermaid >}}

### Who Keeps Track of the Squad?
Enter the RedisDelegationStore. Think of it as the agent equivalent of a team manager. It maps domains like “finance” to whichever agent knows about numbers. The mapping looks a bit like a phone book in Redis, complete with keywords so our supervisor doesn’t misdial.
```python
class RedisDelegationStore:
    DOMAIN_KEY_PREFIX = "delegate:domain:"
    DOMAINS_KEY = "delegate:domains"

    async def register_domain(
        self,
        domain: str,
        agent_id: str,
        keywords: List[str],
        skills: Optional[List[str]] = None,
    ) -> str:
        key = f"{self.DOMAIN_KEY_PREFIX}{domain}"
        data = {"agent_id": agent_id, "keywords": keywords, "skills": skills or []}
        await self.redis.set_value(key, data)
        await self.redis.add_to_set(self.DOMAINS_KEY, domain)
        logger.info(f"Registered delegation for domain {domain} -> {agent_id}")
        return domain
```

### The Supervisor Gets Smart
Instead of juggling every request, the supervisor now passes messages to the right agent. It calls an LLM to figure out which domain (or teammate) should handle the task:

```python
if self.config.is_supervisor and self.delegations:
    matched_agent: Optional[Agent] = None
    domain = await self._determine_domain(user_message)
    if domain and domain in self.delegations:
        candidate = self.delegations[domain].get("agent")
        if candidate and candidate.config.skills:
            matched_agent = candidate
            logger.info(
                f"Delegating message about {domain} to {candidate.config.agent_id}"
            )
        ...
    if not matched_agent and "general" in self.delegations:
        matched_agent = self.delegations["general"].get("agent")
        if matched_agent:
            logger.info(
                f"Delegating message to general agent {matched_agent.config.agent_id}"
            )
    if matched_agent:
        self.conversation_delegates[conversation_id] = matched_agent
        return await matched_agent.process_message(user_message, user_id, conversation_id)
```
If the user is yelling “stock price,” the finance agent leaps into action. Otherwise, the supervisor might hand it to a general agent which might use reasoning to provide a stale price or perform a time consuming operation of using web-search skill to figure out the price from news.

### Codex Experience
 So I started with Codex UI and it was a pleasant experience. As soon as I signed up (_I am a plus user_), it immediately spun up tasks to write tests and fix bugs. Talk about a overenthusiatic employee. It then submitted PRs which I eyeballed and approved. I later setup the environment and with the tests written above, I was becoming more confident. Although it wasn't able to setup docker containers for me to run the platform but it had access to python runtime to run tests. Like Claude Code, it also needed an AGENTS.md file which I copied from CLAUDE.md and enhanced with my multi-agent requirements. After that I was able to pass in the initial prompt and let it wheel away. 

 ### Challeges
 * The PR approach although good, creates mess later. I had to create new PRs or update PRs which didn't sync with my machine for me to test it live.
 * Context can become harder for it to understand later due to compaction IMO. Solution was to spin up a new task
 * As I continue to iterate, it became harder to view the diffs for each prompt. I basically had to push the PR and view the changes in github to see what changes were made.


I didn't use Codex CLI in the beginning since it wasn't able to pick up its performant codex-mini-latest model at the beginning. I did use it at the end and it did improve my experience tremendously. I think CLI experience is going to win 

### Tests, Because We’re Professionals
We added unit tests that monkeypatch everything—including the LLM—so we don’t run up our API bill. These tests confirm that delegation happens correctly and that conversation IDs don’t mysteriously change mid-chat. It’s like a trust fall for your code. I am coming across some TDD practices to keep your vibe-coding sessions sane.
```python
async def dummy_call_llm(messages, **kwargs):
    content = messages[-1]['content'].lower()
    if 'stock' in content or 'share' in content or 'ticker' in content:
        return {'domain': 'finance'}
    return {'domain': None}

supervisor = Agent(
    supervisor_config,
    memory_manager=mem,
    delegations={'finance': {'agent': finance_agent}}
)
out = asyncio.run(supervisor.process_message('What is the current price of AAPL stock?', 'user1'))
assert 'AAPL' in out.message.content

```

With domain‑driven delegation, our platform now behaves like a group chat of specialized bots. The supervisor is basically the team lead, the finance agent is the numbers nerd, and there’s always a generalist around for miscellaneous tasks. Developing it did have a share of challenges. As Tobi Lutke and Andrej Karpathy says - Its about passing context

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">I really like the term “context engineering” over prompt engineering. <br><br>It describes the core skill better: the art of providing all the context for the task to be plausibly solvable by the LLM.</p>&mdash; tobi lutke (@tobi) <a href="https://twitter.com/tobi/status/1935533422589399127?ref_src=twsrc%5Etfw">June 19, 2025</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

The supervisor agent and its delegated agent sometimes jostled to provide an answer either via their reasoning nodes or their incompatible skills. It’s the bot equivalent of a college project where a subset seems to know everything and remaining are just absent.

Eventually everything did land at a PoC Level. I will keep plugging away and in the next iteration even introduce MultiAgent to our favourite framework -  MCP!

PR for introducing MultiAgent - https://github.com/veegit/agent-platform/pull/9

_This article is part of the my experiment of building an Agent Platform, you can view the progress as a part of the series here [Agentic Platform Series](/series/agentic-platform/)_