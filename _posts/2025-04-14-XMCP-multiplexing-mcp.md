---
title: "Meta MCP: Chaining Tools via Prompt-Driven Arguments"
author: cef
date: 2025-04-14
categories: [Technical Writing, Open Source]
tags: [MCP, AI, Open Source]
render_with_liquid: false
description: This post explores the concept of an MCP tool that can chain multiple tools within a single request, where the arguments for each tool can be dynamically generated using prompts based on the outputs of previous tools.
---

> Check out [this video](https://youtu.be/xPq53oQi2tY) for a deep dive into MCP.


<figure>
 <figcaption style="
    font-style: italic;
    font-size: 1em;
    color: #555;
    background-color: #f9f9f9;
    border-top: 1px solid #ddd;
    padding: 8px 12px;
    text-align: center;
  ">Three MCP tools are called in a single MCP request. The arguments for tools 2 and 3 are 'PROMPT_ARGUMENT' and are inferred based on the result of tool 1. </figcaption>
    <video style="width: 100%; height: auto; display: block;" controls>
    <source src="https://github.com/user-attachments/assets/ad264bab-f135-462d-9fed-e8aedf7ffcc7" type="video/mp4">
    Your browser does not support the video tag.
    </video>
    
</figure>

## Intro 
I've been experimenting with Model Context Protocol and learning more by building yet another MCP server. In my case, it's an LLM interface for interacting with Apache Kafka: [`kafka-mcp-server`](https://github.com/CefBoud/kafka-mcp-server).

One thing I noticed, though, is that I often need to call 2 or 3 tools to perform a simple action, where the result of tool 3 depends on the output of tools 1 or 2. Over time, this became quite tedious.

Then I thought—why not multiplex or bundle multiple tool calls together, with arguments as `PROMPT_ARGUMENT`s that get resolved after the previous tools have run? For example:

1. List the topics present in the cluster.  
2. Read messages from the topic related to transactions.  
3. Create a duplicate of that topic named `${originalName}-dup`.

Workflows like this—or any others where results can be easily extracted but require too much back-and-forth—become much simpler with this new multiplexing tool.

And this multiplexing tool is really just a simple utility that takes an array of tool call requests:

```json
{
        "description": "Takes a list of tool requests and executes each one, returning a list of their results. Use this tool when you need to call multiple tools in sequence. If an argument for a tool at position N depends on the result of a previous tool [1...N-1], you can express that argument as a prompt to the LLM using the format `PROMPT_ARGUMENT: your prompt here`. For example: `PROMPT_ARGUMENT: the ID of the created resource.`",
        "inputSchema": {
          "type": "object",
          "properties": {
            "tools": {
              "description": "List of tool requests",
              "items": {
                "properties": {
                  "id": {
                    "description": "the request ID.",
                    "required": true,
                    "type": "number"
                  },
                  "jsonrpc": {
                    "description": "jsonrpc version.",
                    "required": true,
                    "type": "string"
                  },
                  "method": {
                    "description": "The MCP Method. 'tools/call' in this case.",
                    "required": true,
                    "type": "string"
                  },
                  "params": {
                    "properties": {
                      "arguments": {
                        "description": "The tool's arguments derived from each specific tool input schema.",
                        "required": true,
                        "type": "object"
                      },
                      "name": {
                        "description": "The tool's name.",
                        "required": true,
                        "type": "string"
                      }
                    },
                    "type": "object"
                  }
                },
                "type": "object"
              },
              "type": "array"
            }
          },
          "required": [
            "tools"
          ]
        },
        "name": "MultiplexTools"
      }
```

My implemetation is rather short and boils down to [this tiny snippet](https://github.com/CefBoud/kafka-mcp-server/blob/main/pkg/kafka/multiplex.go):
```go

func BeforeToolCallPromptArgumentHook(ctx context.Context, id any, message *mcp.CallToolRequest) {
	messageJson, _ := json.Marshal(message)
	toolContext := addStringToToolCallContext("Tool call request:\n" + string(messageJson))

	for k, v := range message.Params.Arguments {
		if arg, ok := v.(string); ok {
			strings.HasPrefix(arg, "PROMPT_ARGUMENT")
			prompt := InferArgumentPrompt(toolContext, arg)
			inferredArg, err := QueryLLM(prompt, getModel(ctx))
			if err != nil {
				return
			}
			message.Params.Arguments[k] = inferredArg
		}

	}
}

func AfterToolCallPromptArgumentHook(ctx context.Context, id any, message *mcp.CallToolRequest, result *mcp.CallToolResult) {
	fmt.Fprintf(os.Stderr, "inside AfterToolCallPromptArgumentHook %v", message)
	resultJson, _ := json.Marshal(result)
	addStringToToolCallContext("Tool call result:\n" + string(resultJson))
}

tools := request.Params.Arguments["tools"].([]interface{})
			var result []any
			for _, tool := range tools {
				payload, _ := json.Marshal(tool)
				response := s.HandleMessage(ctx, payload)
				result = append(result, response)
			}
			jsonResult, _ := json.Marshal(result)
			return mcp.NewToolResultText(string(jsonResult)), nil
```

## Conclusion

This has been a lot of fun. One idea that came to mind is adding some kind of Lua interpretation to run tools based on certain Lua conditions or to compute specific arguments, instead of relying solely on the LLM.

Another one is to introduce branching logic—essentially building a DAG or something similar to Airflow or Spark, where workflows can follow different paths based on conditions.

This really showcases the model's intelligence and is akin to an algorithm where the MCP client LLM orchestrates steps using the tools and prompt arguments.

---

Thanks for getting this far! If you want to get in touch, you can find me on Twitter/X at [@moncef_abboud](https://x.com/moncef_abboud).  
