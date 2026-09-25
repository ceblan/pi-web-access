# Feature Registration Gates

Allows disabling specific tools and commands via `web-search.json` config without code changes.

## Per-tool gates

Tools can be individually enabled/disabled via `tools.<toolName>.enabled` in the config. When disabled, the tool is not registered with pi at all.

Supported keys: `webSearch`, `sourceCheck`, `fetchContent`, `getSearchContent`.

### Example config

```json
{
  "tools": {
    "sourceCheck": { "enabled": false },
    "getSearchContent": { "enabled": false }
  }
}
```

Result: only `web_search` and `fetch_content` are registered.

## Command gates

Commands can be disabled via `commands.<name>.enabled` in the config.

Supported keys: `websearch`, `curator`, `search`, `google-account`.

## Image config

Controls whether images are fetched during content extraction via `image.enabled` in the config.

See [[feature-config.ts#canAttachImages]].

## Image extraction gate

Short-circuits image fetching in `extractViaHttp` and frame/timestamp extraction before any hosted provider fallback.

See [[extract.ts#imageGateError]].

## PDF gate

When `pdf.enabled: false` is set in the config, PDF extraction is short-circuited before any hosted provider fallback is attempted.

See [[pdf-extract.ts#loadPDFConfig]].

## Legacy support

The legacy `webSearch.enabled: false` config still works — it disables `webSearch` and `sourceCheck` while leaving `fetchContent` and `getSearchContent` enabled.

The new per-tool gates take precedence over the legacy flag when both are specified.
