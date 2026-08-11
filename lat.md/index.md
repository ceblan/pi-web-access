# pi-web-access

Web search and content extraction extension for pi coding agent.

## Features

Per-tool and per-command enable/disable via config; tool schemas advertise only configured providers.

### Feature Registration Gates

See [[features#Feature Registration Gates]].

### Provider Advertising

See [[providers#Provider Advertising]].

## Architecture

Search providers are resolved and routed at runtime via [[gemini-search.ts]]. Content is extracted via [[extract.ts]] with PDF and image support gated by [[feature-config.ts]].

## Configuration

Configuration is loaded from `web-search.json` via [[utils.ts#getWebSearchConfigPath]].
