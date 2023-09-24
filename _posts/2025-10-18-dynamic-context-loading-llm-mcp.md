---
title: "Reducing Context Bloat with Dynamic Context Loading (DCL) for LLMs & MCP"
author: cef
date: 2025-10-04
categories: [Technical Writing, Open Source]
tags: [MCP, AI, LLM]
render_with_liquid: false
description: "DynamicContextLoading is a lightweight technique for on-demand tool activation in LLMs and MCP, reducing context bloating by loading tools only when needed."
---



[DynamicContextLoading Github Repo with code examples](https://github.com/CefBoud/dynamiccontextloading)

## Intro

While working on [Moncoder](https://github.com/CefBoud/MonCoder/), a simple coding agent, I noticed the context was getting cluttered with rarely used tool calls, like loading an entire database into RAM just to read a few rows. That led to [**Dynamic Context Loading**](https://github.com/CefBoud/dynamiccontextloading): instead of preloading every tool, the model gets a quick summary of available capabilities and only loads specific tools into context when needed. This keeps things lean and efficient.

It works with MCP as well, where multiple MCP servers can quickly eat up context space.

When the number of tools or servers grows, you can add multiple loading levels—like cache tiers. The model starts with a high-level module (MCP server) summary, drills down to tool summaries if needed, and only loads full tool definitions when it's ready to use them. All this is handled with a simple `loader` tool and my testing suggests LLMs understand this pattern rather well.



## The Problem of Context Bloating
LLMs have limited context windows, causing issues when all tools are included in every interaction: higher API costs, slower responses, reduced accuracy, and potential token limit errors. DynamicContextLoading tries to address this by activating tools on-demand, maintaining a lean context.

## Principles
- **On-Demand Activation**: Tools are loaded into context only when requested via the loader.
- **Summary Generation**: Brief descriptions are generated dynamically from full tool definitions, having the models generate the summaries is useful because it can pick up on nuances and differentiate similar tools, the model will rely on its own understanding to pick a tool later.

## The Gist of it


```python
TOOL_REGISTRY = ...  # Central registry holding all available tools

def create_loader_tool():
    # Creates a special tool that activates other tools by name
    briefs = generate_summaries(TOOL_REGISTRY)
    description = f"Loader tool. Available tools: {briefs}"

    def loader_execute(tool_names):
        # Adds selected tools to active list for LLM context
        for name in tool_names:
            if name in TOOL_REGISTRY:
                active_tools.append(TOOL_REGISTRY[name].definition)
        return f"Activated: {tool_names}"

    return Tool({"function": {"name": "loader", "description": description}}, loader_execute)

# Conversation loop: LLM interacts, calls tools dynamically
def conversation_loop():
    messages = [{"role": "user", "content": "User query here"}]
    active_tools = []  # Start with no tools active
    while True:
        # active_tools is a dynamic list that the loader enriches when asked by the LLM.
        response = llm_completion(messages, tools=active_tools) 
        if response.tool_calls:
            for tool_call in response.tool_calls:
                func_name = tool_call.function.name
                args = tool_call.function.arguments

                if func_name == "loader":
                    # Activate tools via loader
                    tool_names = args.get("tool_names", [])
                    result = loader_tool.function(tool_names)
                    messages.append({"role": "tool", "content": result})
                else:
                    # Execute other tools
                    tool = TOOL_REGISTRY.get(func_name)
                    if tool:
                        result = tool.function(**args)
                        messages.append({"role": "tool", "content": result})
        else:
            # Final response from LLM
            print(response.content)
            break

# In use: Register tools, create loader, run conversation
active_tools = []  # Global active tools list
loader_tool = create_loader_tool()
active_tools.append(loader_tool)
conversation_loop()
```

the LLM starts only the loader tool which has a summary of all tools (just a brief summary, no bloat). Whenever the LLM is interested in some tool, it calls the loader which add it to the `active_tools`. 
**Sample Interaction:**
```
User: I need to calculate 15 * 7 and then get the weather in New York.

Adding the following tools to context: calculator, get_weather

*Executing tools..*

15 * 7 equals 105 and thhe weather in New York is cloudy with a temperature of 8°C.
```

## Example with MCP
Dynamic loading integrates with MCP servers (e.g., GitHub MCP via stdio) by listing available tools, generating briefs from their definitions, and activating them on demand via the loader tool to maintain a lean context. With multiple MCP servers, we include only a couple of sentences per server instead of their entire tool list. More details are added to the context on a need-to-know basis.

### Multi-Level Dynamic Context Loading with MCP
The system uses a two-level loading mechanism to manage MCP tools efficiently:

1. **Server Descriptions (Level 1)**: At startup, only high-level server descriptions are loaded into the context (e.g., "GitHub MCP: Provides tools for GitHub management"). This gives the LLM awareness of available servers without bloating the context with all tool details.

2. **Tool Summaries (Level 2)**: When the LLM needs more details, it can load summaries for a specific server's tools using the 'load_tool_summaries' action. This adds brief descriptions of each tool (e.g., "get_me: Retrieves details of the authenticated GitHub user.") to the context, allowing informed decisions on which tools to activate.

3. **Actual Tools (Level 3)**: Finally, specific tools are loaded into the active context using the 'load_tools' action, making them callable for the current interaction. This ensures only necessary tools are present, keeping the context lean.

## Example Run


```sh
# Run the script (assumes .env with GITHUB_PERSONAL_ACCESS_TOKEN set)
uv run dcl_mcp.py

# Output shows MCP servers initializing and listing tools
load_mcp_tools github ['add_comment_to_pending_review', 'add_issue_comment', ...]
load_mcp_tools figma ['get_figma_data', 'download_figma_images']

# Initial loader description: Only server-level info loaded
generated loader tool description: Dynamic Tool Loader for managing MCP tools...
'github' MCP: The GitHub MCP server provides tools for comprehensive GitHub management...
'figma' MCP: This MCP server enables fetching detailed Figma file data...

# User query triggers loader activation
User: Provide a list of 5 of my public GitHub repositories

# LLM uses loader to load GitHub tool summaries (Level 2)
tool call: {"action":"load_tool_summaries","servers":["github"]}
Loader result: Loaded tool summaries for servers: github.

# Now context includes tool briefs for GitHub
Tools summaries loaded:
- get_me: Retrieves details of the authenticated GitHub user.
- search_repositories: Searches for repositories by name, topics, or metadata.
[... other tools ...]

# LLM activates specific tools (Level 3)
tool call: {"action":"load_tools","tools":["get_me","search_repositories"],"server":"github"}
Loader result: Activated tools from github: get_me, search_repositories.

# Tools now active and callable
active tools: loader, get_me, search_repositories

# LLM calls get_me to get user info
tool call: {} (for get_me)
get_me result: {"login":"CefBoud", ...}

# Then calls search_repositories with query
tool call: {"query":"user:CefBoud","sort":"updated"} (for search_repositories)
search_repositories result: List of 5 public repos (e.g., MonCoder, cefboud.github.io, etc.)

# Final response from LLM
Assistant: Here is a list of 5 of your public GitHub repositories...
```


