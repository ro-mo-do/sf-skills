# sf-skills

A collection of AI assistant skills for Salesforce development, built and maintained by the FDE team.

Each skill is a self-contained prompt package — a structured `SKILL.md` that guides any AI assistant through setup, configuration, and development workflows within a specific Salesforce domain. Skills work with Claude Code, Cursor Agent, or any AI assistant that can read context files.

## Skills

| Skill | Description |
|---|---|
| [sf-hosted-mcp](sf-hosted-mcp/) | End-to-end setup for Salesforce Hosted MCP Servers — ECA creation, server activation, and client configuration for Claude Code, Cursor, and Postman |

## Usage

### Claude Code (recommended)

Install all skills at once:

```bash
claude skills install https://github.com/ro-mo-do/sf-skills.git
```

Or install a single skill:

```bash
claude skills install https://github.com/ro-mo-do/sf-skills.git/sf-hosted-mcp
```

Then invoke by name from the Claude Code prompt:

```
/sf-hosted-mcp
```

### Other AI assistants (Cursor Agent, ChatGPT, etc.)

Clone or download this repo, then paste the contents of the relevant `SKILL.md` into your assistant as context — either as a system prompt, a file attachment, or by referencing it in your prompt:

```
Use the instructions in sf-hosted-mcp/SKILL.md to help me connect to Salesforce via MCP.
```

## Contributing

This repo is maintained by the FDE team. To add a new skill:

1. Create a subdirectory named `sf-<topic>/`
2. Add a `SKILL.md` (the skill prompt) and a `README.md` (human-readable overview)
3. Add any supporting assets to an `assets/` subdirectory
4. Add a row to the Skills table above
5. Open a PR for review

## License

MIT
