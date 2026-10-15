# Bookstore

Basic app using Java 11 and MongoDB.

It requires a bookstore database to be created in MongoDB.

This API manages book inventory in a bookstore. It supports listing, retrieving, creating, and updating books.

## API Specification

The OpenAPI 3.0 specification for the endpoints currently implemented by the service is available at [docs/openapi.yaml](docs/openapi.yaml).

The API base path is `/api`. The specification documents current behavior, including response statuses and known limitations; it does not imply that unimplemented operations such as deleting a book are available.

## Technical Documentation

See [docs/technical-documentation.md](docs/technical-documentation.md) for the architecture, project structure, MongoDB integration, runtime behavior, and build/test instructions.

## Optional: Use docker compose to run the application

```bash
docker-compose up
``` 

# Harness Architecture

## Tools

| Tool | Why | Install (reference) |
|------|-----|---------------------|
| **GitHub MCP** | Fetch issues (included in `.mcp.json`) | Uses Copilot GitHub auth — `https://api.githubcopilot.com/mcp/` with `X-MCP-Toolsets: issues` |
| **CodeGraph** | Code navigation (base) | `npm i -g @colbymchenry/codegraph` → `codegraph install` → in repo `codegraph init` |
| **RTK** | Compress shell/test noise | `brew install rtk-ai/tap/rtk` (or Cargo from `rtk-ai/rtk`) → `rtk gain` to verify → `rtk init -g --copilot` |
| **Caveman** | Chat prose lite only | `npx skills add JuliusBrussee/caveman -g` → use `/caveman lite` in chat |
| **Graphify** (optional) | Project/docs/schema map | Install per [Graphify](https://github.com/safishamsi/graphify), then add the MCP block below |
| **Spec Kit** (optional) | Spec-driven delivery | [github/spec-kit](https://github.com/github/spec-kit) `specify init` for Copilot |
| **grill-with-docs** (optional) | ADRs / CONTEXT alignment | From [mattpocock/skills](https://github.com/mattpocock/skills) |

### Optional Graphify MCP

Add only if Graphify is installed. Portable `.mcp.json`:

```json
"graphify": {
  "type": "stdio",
  "command": "graphify",
  "args": [".", "--mcp"]
}
```

VS Code `.vscode/mcp.json` uses the same server entry under `"servers"`.

## How Harness is configured

- **GitHub MCP** to load an issue before work  
- **CodeGraph** for code structure  
- **Graphify** for project/docs (if present)  
- **RTK** for noisy shell  
- **Caveman lite** for chat only — never for SPEC/ADR/PR text  
- **Spec Kit** alone as the delivery spine if you use one  

No phase profiles. No parallel Superpowers + Spec Kit.
