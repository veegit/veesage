+++
title = 'Playing with ChatGPT to enhance the Agentic Platform'
date = 2025-05-19T20:55:05-07:00
+++

Now that I was able to build the [basic backend platform to build and serve agents](/articles/playing-with-claude-code/) via Claude Code, I wanted to now build a frontend to interact with the agents.

I chose to use recently launched Jules from Google along with tried and tested ChatGPT Plus (another one of many subscription-based services I pay for)

My trial with Jules suffice to say was pretty disappointing. The wait times were long and the code output wasn't that great. I then had to resort to the ChatGPT Plus to continue building the UI.

It was pretty easy, just hook up my repository and enable DeepResearch and give the requirements

```
Generate a web chat UI which can call the conversation API. 
Requirements

* Web UI starts by asking a userId and a drop down of Agent to select 
* Web UI then starts a conversation using POST /conversations API with userId, agentId and first message
*  Web UI then waits for a response and show a thinking icon to let user wait. Ensure during that time, the send button is disabled
*  Web UI then shows the response and stores the returned Id as the conversation_id and stores it for the session
* Web UI can allow more message to be sent using POST /conversations/{conversation_id}/messages API and user can continue to chat
* Web UI should keep the names user_id and agent name as user talks with the agent

These are the different Urls

Conversation Service: http://localhost:8000 - Main API for client applications 
Agent Lifecycle Service: http://localhost:8001 - Manages agent configuration 
Skill Service: http://localhost:8002 - Manages skills and skill execution 
Agent Service: http://localhost:8003 - Runs agent workflows 

This is how you start a conversation 
curl -X POST http://localhost:8000/conversations -H "Content-Type: application/json" -d '{ "agent_id": "YOUR_AGENT_ID", "user_id": "user123", "initial_message": "Tell me about artificial intelligence" }'

This is how you continue the conversation
 curl -X POST http://localhost:8000/conversations/YOUR_CONVERSATION_ID/messages -H "Content-Type: application/json" -d '{ "content": "What are the latest developments in machine learning?", "user_id": "user123" }'

This is how you list all agents curl -X GET http://localhost:8003/agents

Ensure the UI is callable and host it as a separate service in docker.
```

<br>

This did generate a basic UI for me but I had to give many further prompts to get it to right. E.g.

>Why would I need another DockerFile. I already have one here in the same root directory as docker-compose.yaml

>Show the code for  style.css and script.js here in the chat

>Its failing in python with
>api-1              |   File "<frozen importlib._bootstrap>", line 241, in _call_with_frames_removed

>I don't see a UI to ask for user_id and dropdown of agents

>Good but how do I ensure any changes to frontend assets like css, html and js is immediately available without redeploying the docker

>While starting the initial conversation it fails with
>POST http://localhost:8000/conversations 405 (Method Not Allowed)

After some trial and error, I got the UI working. I did have to resort to some coding myself to get the whole experience working end to end but it was a fun experience. I did get some kick of of chatting with my spawn :)

<br>

Some challenges while working with Jules
1. Wait times were long like in the range of 30-60 minutes
2. Code quality was not that great and was coding like an intern who has only done coursework.

<br>

Some learnings while working with ChatGPT Plus
1. Be elaborate in your requirements. Provide schema for responses, API ports etc. 
2. You HAVE to continue to use deep research mode to get the best of out it. I tapped out of Deep Research after first message to save on 10 limit I have.
3. Remind it to add the code inline in Chat. Sometime the Canvas mode doesn't format/highlight the code properly
4. Probe the code output and ask follow-up questions to allow it to correct itself. 

![UI for Agent Platform](/images/playing-with-chatgpt.gif)