---
title: "Is MCP Worth the Hype?"
author: cef
date: 2025-12-15
categories: [Technical Writing, Open Source]
tags: [AI, MCP, LLM]
render_with_liquid: false
description: "Saying that Model Context Protocol (MCP) adoption has been explosive is almost an understatement. This post examines MCP and the problem it solves, while also digging into the issues it faces."
social_preview_image: /assets/mcp-which-way.png
---
![MCP Which Way](/assets/mcp-which-way.png)


On November 25th (which happens to be the birthday of yours truly) 2024, Anthropic [announced the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol). 

MCP is popular, very popular. The [MCP servers](https://github.com/modelcontextprotocol/servers) repository has 74k+ stars. For a repo that hosts reference implementations of a spec, that's quite an achievement. World-class companies have built MCP servers: AWS, GitHub, Notion, Figma, and Slack, to name a few.
The amount of excitement around MCP has been great. A lot of people are, however, skeptical. 

Seeing that [Anthropic just donated](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation) the project to the freshly minted Agentic AI Foundation, I decided to write this post in which I'll attempt to synthesize the arguments for and against MCP.

## The Case for MCP

If you have 45 min to spare and are willing to humor my ramblings, I made a [video](https://www.youtube.com/watch?v=xPq53oQi2tY&t=2484s) that goes through the spec in a detailed manner.

If not, in a nutshell, MCP is a standard that allows LLM apps to interact with different servers/providers using a common, agreed-upon format. I can use any LLM (ChatGPT, Claude, Gemini) to interact with various apps (Figma, Slack, GitHub) using the same format. And it's plug-and-play. I just configure the server (a URL with an optional token) and then I am ready to go!

<details markdown="1">
<summary>
A more detailed explanation if you're up for it.
</summary>

LLMs, in their basic usage, only generate tokens, they can't directly interact with the outside world.
**Function calling** (or **tool use**) enables LLMs to *act* beyond text generation.

At a high level, tool calling is a simple trick:

You include a list of tool/function definitions (names, arguments, schemas) as part of the model's prompt, and instruct the model to output a specific structured response when a tool is relevant. When the model does so, the program running the model executes the tool and feeds the result back into the model's context.

A very simplistic example:

* Tool definition (given to the model):

```json
{
  "name": "get_weather",
  "parameters": { "city": { "type": "string" } }
}
```

* User: Weather in Paris?
* Model: [tool call] `{ "name": "get_weather", "arguments": { "city": "Paris" } }` [/tool call]
* Runtime (Program): runs the function and adds the result (`18°C, cloudy`) to the model's context.
* Model: It's 18°C and cloudy in Paris.

Clever trick. The model is simply a text-outputting machine. But by assigning special meaning to certain sequences and having the runtime wait for them and perform actions, we enabled our model to act.

Ok, what does this have to do with MCP? If you check the documentation that OpenAI, Anthropic, and Google have for their respective models (GPT, Claude, and Gemini), you'll unfortunately find different ways to define functions and call them. This is no good.

If you want to build an app or service that offers tools to LLMs. Say I'm Atlassian and I want my customers to interact with their Jira account (create a ticket, comment on another one, etc.) using their LLMs. I'll build a server that performs tool calls for these actions. Do I have to support all LLM function-call formats? Just the popular ones? Create my own superior format?

That's the first problem.

The second issue is discoverability. In the workflow described above, you need to know the available functions beforehand. It'd be nice to have a mechanism for an LLM-enabled app to send a discovery query asking the servers it wants to use which functions/tools they offer and how to use them.

Then we have a bunch of small nice-to-haves:

* Allowing LLMs to discover and query useful resources/data: user preferences, museum info for a trip, an HTML page to display nicely (this is part of how ChatGPT apps work).
* Allowing the server to ask the LLM/client for things, maybe some user input or giving the LLM-less server access to LLM capabilities, i.e. performing an intermediate task that requires AI help.
* Agreeing on security aspects: proving my identity to Jira when I try to access my account (authentication), and whether I have the right to perform certain actions via the LLM (authorization).

Well, you guessed it. MCP addresses all these points and then some.

</details>

<br>

The [OpenAI Apps SDK](https://developers.openai.com/apps-sdk/) is, in my opinion, its most impressive application of MCP. I did some [exploring](https://youtu.be/YdEK6M0dOmM?si=if_lAfcthZ5h12f4) and I think it has serious potential.

So the MCP promise is enticing: connect any tool to AI apps with a few lines of code. Interact with your digital life through an LLM by having these [discoverable servers](https://registry.modelcontextprotocol.io/). Add a thin layer to your services and have LLMs and agents integrate seamlessly with your ecosystem.

I would say, on the face of it and for the most part, that promise is almost true. But that *almost* is everything. Something about noses being shorter.

## The Case Against

A recurring complaint with MCP is context bloat. If you have 10 MCP servers (not unheard of) and each one has an average of 5 tools, that's 50 tool definitions that your model has loaded in its context, in addition to its own tools. In the case of a coding agent, [these can be numerous](https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/#tools). So you're losing context, arguably penalizing the performance of your model and paying more for your LLM usage, even though you end up using only a very small subset of the MCP tools in practice. This is an approach that will definitely not scale with current model capabilities.
I experimented with a way to fix that using [dynamic tool loading](https://cefboud.com/posts/dynamic-context-loading-llm-mcp/). Anthropic introduced a [Tool Search tool](https://www.anthropic.com/engineering/advanced-tool-use), I haven't played with it, but this might be a fix for the context bloat issue. However, it's not part of the MCP specification.

Another major issue with MCP is security. This is a new, fast-evolving standard. Move fast and break things, I've heard. But with that, vulnerabilities loom. The Figma MCP server had [an RCE](https://thehackernews.com/2025/10/severe-figma-mcp-vulnerability-lets.html) that got a lot of attention. LLMs and MCP offer flexibility and a great deal of power. But ... SpiderMan's cousin said something about power and responsibility. This is not specific to MCP and likely applies to any fast-evolving, widely adopted technology. MCP doesn't invent these risks, but by standardizing and lowering the barrier to powerful integrations and action, it can amplify their impact when things go wrong. Also, with the AI driving and possibly going unchecked, the damage can get scary!

Regarding security, MCP builds on top of [OAuth 2.1 to handle authorization](https://cefboud.com/posts/mcp-oauth2-security-authorization/). From what I can tell, the integration of OAuth has its own set of challenges and isn't completely streamlined yet. One pain point today is client registration. MCP servers often need to support dynamic or semi-dynamic clients (agents, local tools, ephemeral apps), which doesn't map cleanly onto OAuth's traditional pre-registered client model. This should be a solvable problem, and the MCP community will probably refine it and land on a more robust solution. 

The critique of MCP that I disagree with goes along the following lines:

> just use OpenAPI or REST.

It's true, LLMs can do that today. But an API for a production-grade system with very tricky parameters is different from a tool that you expose to an LLM. The model needs end-to-end tools and doesn't have to know your business domain and concepts, unlike a REST API that often offers a great deal of detail and flexibility.

The final critique is that MCP is over-standardization at its finest and that it's too early to sing the praises of this nascent standard. A [comment on HN](https://news.ycombinator.com/item?id=46208501) echoes this perfectly:

> It feels far too early for a protocol that's barely a year old with so much turbulence to be donated into its own foundation under the LF.
> A lot of people don't realize this, but the foundations that wrap up to the LF have revenue pipelines that are supported by those foundations' events (like KubeCon bringing in a LOT of money for the CNCF), courses, certifications, etc. And, by proxy, the projects support those revenue streams for the foundations they're in. The flywheel is *supposed* to be that companies donate to the foundation, those companies support the projects with engineering resources, they get a booth at the event for marketing, and the LF can ensure the health and well-being of the ecosystem and foundation through technical oversight committees, elections, a service desk, owning the domains, etc.
>
> I don't see how MCP supports that revenue stream, nor does it seem like a good idea at this stage: why get a certification for "Certified MCP Developer" when the protocol is evolving so quickly and we've yet to figure out how OAuth is going to work in a sane manner?
>
> Mature projects like Kubernetes becoming the backbone of a foundation, like it did with CNCF, make a lot of sense: it was a relatively proven technology at Google that had a lot of practical use cases for the emerging world of cloud and containers. MCP, at least for me, has not yet proven its robustness as a mature and stable project. I'd put it in the "sandbox" category of projects that are still rapidly evolving and proving their value. I would have much preferred for Anthropic and a small strike team of engaged developers to move fast and fix a lot of the issues in the protocol versus it getting donated and slowing to a crawl.

The whole thread is insightful, and people seriously comparing MCP to K8s feels wrong. MCP is a thin, simple layer that you add on top of your code to make it LLM-friendly, and it rubs some people the wrong way that others are getting carried away with it. It gives credence to the "AI Bubble" thesis and understandably so.


## Closing Thoughts



I think MCP's clear usefulness and ease of use made it a great way for programmers to dip a foot into AI. That was the case for me.

It's also very easy to build things with. As a result, it became a perfect way for companies to jump on the AI train and showcase their AI adoption. Slap a couple of decorators on your API endpoints and your senior executives can proclaim: *we're adopting AI and are on the bleeding edge*. I think this fueled the hype and, in turn, gave MCP a bad rep.

But I still think MCP is great. It solves a clear set of problems and enables quick and easy adoption of AI. Having a standard, even a "thin" one, is a powerful thing: conversations start. MCP-UI and ChatGPT apps are a perfect example of this.

But how simple is too simple?  Looking at you, [agents.md](https://agents.md/).

MCP has its fair share of problems, most of which are actively being addressed. I hope it'll have a bright future and enable more people to build and prosper with AI.

<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/CY9ycB4iPyI"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>