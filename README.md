# Agent playground

This repository explores a generic agentic system for OHDSI analytics.
It uses the (emerging) standards for agentic workflows using the `.agents` folder.
Unfortunately, tool definitions (MCP services) currently are not standardized, so we include multiple platform-specific definitions.

This repository has been successfully tested with:

- GitHub Copilot in Visual Studio Code (uses `.vscode/mcp.json`)
- OpenCode (uses `opencode.json`)

The MCP services are not recognized in PyCharm.

## Examples

Clone this repo, open it in VSCode with GitHub Copilot chat or OpenCode, and type the suggested prompts. Make sure your chat is in *agent* or *build* mode.

## Question standardizer


Use free text (let the system find the relevant skill):

```
I want to study whether naproxen causes MI. Structure following OHDSI analysis templates.
```

Or explicitly invoke the skill:

```
/ohdsi-question-standardizer Does naproxen cause MI?
```

Or invoke the skill on a file. You can use the protocol of the first OHDSI network study in in `examples/`:

OpenCode:
```
/ohdsi-question-standardizer @examples/OHDSI treatment patterns 30nov2014.md 
```

GitHub Copilot:
```
/ohdsi-question-standardizer #OHDSI treatment patterns 30nov2014.md 
```


## Hecate vocab search

```
What standard concepts denote bipolar disorder?
```

