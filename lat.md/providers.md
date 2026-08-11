# Provider Advertising

Narrows the provider catalog announced to the LLM in tool schemas to only the configured providers.

## Motivation

Before this change, tool schemas enumerated all 21+ supported providers. This added ~447 tokens/session to the system prompt.

## Design

Static approach: `EXPOSED_RESOLVED_PROVIDERS` and `EXPOSED_SEARCH_PROVIDERS` in [[index.ts]] are consumed only by `searchProviderSchema()`. Runtime validation and routing remain unchanged.

Exposed providers: `searxng`, `tavily`, `perplexity`, `gemini` + `auto`/`all`.

## Trade-offs

Tool calls with an explicitly unlisted provider are rejected at schema validation. However, `auto` and `all` still route to any available provider at runtime.

## Tool Gates vs Advertising

Disabling `sourceCheck` via `tools.sourceCheck.enabled: false` saves ~262 additional tokens since the tool is not registered at all.
