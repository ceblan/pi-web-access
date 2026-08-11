# Plan de Implementación — Strip de providers en `pi-web-access` (fork `ceblan`)

## Goal

Reducir ~450–700 tok/sesión del coste fijo de `pi-web-access` exponiendo en los schemas/descripciones **únicamente los providers configurados** (`searxng`, `tavily`, `perplexity`, `gemini` + `auto`/`all`), sin tocar la extracción de contenido (firecrawl) ni la semántica de `auto`/`all`.

## Estado del setup (parte 1 y 2 del encargo)

- ✅ Clone en `~/git-carlos/pi-extensions/pi-web-access` @ `7a3357c` (= `v0.20.0` + 2 commits: gates de tools + docs). Remote actual: solo `upstream → git@github.com:nicobailon/pi-web-access.git`.
- ⚠️ **BLOQUEANTE**: el fork `ceblan/pi-web-access` NO existe. El token de `gh` es un **fine-grained PAT** (`github_pat_…`) → `gh repo fork` devuelve HTTP 403 (estos tokens no pueden forkear repos de terceros). Se requiere acción del operador (Task 0).

## Datos establecidos (verificados en esta sesión)

| Hecho | Detalle |
|---|---|
| Providers configurados (`~/.pi/web-search.json`) | `searxngBaseUrl`, `tavilyApiKey`, `perplexityApiKey`, `geminiApiKey`; `provider: "searxng"` (default) |
| Firecrawl | `firecrawlBaseUrl/ApiKey/ApiVersion/FreshScrape` — usado SOLO en `firecrawl.ts`, `extract.ts`, `brightdata.ts`, `brightdata-unlocker.ts` (extracción de contenido). **No es un provider de búsqueda; el strip no lo toca.** |
| Constantes | `gemini-search.ts:29-30` — `RESOLVED_SEARCH_PROVIDERS` (21) / `SEARCH_PROVIDERS` (23); `:93` `ALL_SEARCH_PROVIDERS` (16) para `provider:"all"`, filtrado en runtime por disponibilidad (`:389`) |
| Schema | `index.ts:230 searchProviderSchema()` → usado SOLO 2×: `web_search` (`:1618`) y `source_check` (`:2178`). `fetch_content`/`get_search_content` NO tienen param provider |
| Descripción `web_search` | `index.ts:1604-1606` — enumera 21 providers 2× (lista principal + cadena fallback) |
| Cadena `auto` | `search()` en `gemini-search.ts:481+`: hardcodeada con availability checks (SearXNG primero si configurado; además `provider:"searxng"` en config cortocircuita a searxng). **Independiente de los enums** |
| Gates de tools | Commit `7a3357c` (HEAD, NO está en el 0.20.0 instalado): `tools.<nombre>.enabled` en web-search.json permite desactivar `source_check`/`fetch_content`/etc. **sin tocar código** |
| Typecheck | `providerLabel` (`gemini-search.ts:356`) y helpers comparan contra los 21 literales → **estrechar `RESOLVED_SEARCH_PROVIDERS` rompería `tsc`** (TS2367). Descartado |
| Tests | `test/` cubre runtime de providers (brightdata, jina, tinyfish, precedence…), no snapshots del schema → el strip "advertise-only" no rompe tests |
| Instalación actual | `~/.pi/agent/settings.json` → `"npm:pi-web-access"`; `~/.pi/agent/npm/package.json` → `"pi-web-access": "^0.20.0"` |

### Medición antes/después (gpt-tokenizer o200k_base, `/tmp/tb/strip-measure.mjs`)

| Elemento | Antes | Después | Δ |
|---|---:|---:|---:|
| `web_search` description | 374 | 179 | **−195** |
| `web_search` provider param (schema union) | 232 | 106 | **−126** |
| `source_check` provider param (schema union) | 213 | 87 | **−126** |
| **Subtotal strip** | **819** | **372** | **−447** |
| (+) desactivar `source_check` vía gate (tool completo ≈389) | — | — | **−262 adicionales** |

Ahorro total esperado: **~447 tok/sesión** (solo strip) o **~709** (strip + gate de `source_check`). La estimación del brief (~700–900) se alcanza con la combinación.

## Decisión de diseño: estático vs dinámico

| | **Estático "advertise-only" (RECOMENDADO)** | Dinámico (enum desde config al registrar) |
|---|---|---|
| Tokens ahorrados | Idénticos (~447) | Idénticos (~447) |
| Diff | ~15 líneas nuevas + 3 strings | Generador de enum + templating de descripción + mapeo key→provider de 21 providers + fallback si config vacía |
| Riesgo typecheck/tests | Nulo (runtime y validación intactos) | Bajo-medio (timing de lectura de config, config ausente) |
| Añadir un provider después | Editar 1 array en el fork | Automático |
| Merge de `upstream` | Conflictos casi nulos (constantes nuevas propias + hunks delimitados) | Similar |
| Providers no listados | **Bloqueados en tool calls explícitas**: el enum reducido del JSON schema rechaza `provider:"brave"` en validación, antes de `execute()` (intencional). Matices verificados: (a) el runtime queda intacto — tool calls con provider omitido/`auto`/`all` sí pueden enrutar a providers no expuestos si la config/disponibilidad los selecciona (esos valores sí están en el enum); (b) los 23 siguen aceptándose en rutas **no-tool**: comando `/websearch`, picker del curator y key `provider` de web-search.json (`normalizeProviderInput`, `index.ts:291` → lista completa) | Depende de implementación |

**Recomendación: estático advertise-only.** La config del usuario es estable (4 providers); el dinámico añade complejidad y modos de fallo sin ahorro extra. Variante clave respecto a la idea original: **NO** modificar `RESOLVED_SEARCH_PROVIDERS`/`SEARCH_PROVIDERS` (rompería `tsc` y tests); añadir constantes `EXPOSED_*` consumidas únicamente por `searchProviderSchema`. Consecuencia deliberada y verificada: en tool calls (`web_search`, `source_check`) un provider **explícito** no expuesto es rechazado por la validación del JSON schema — el schema es la frontera de lo anunciable; el enrutado runtime de omitido/`auto`/`all` (basado en config + availability checks) conserva la capacidad completa.

`source_check`: **mantenerlo** con el enum reducido (213→87). Opcional adicional: gate `tools.sourceCheck.enabled=false` (−262 más) — decisión explícita del usuario, no parte del strip por defecto.

## Tasks

0. **[OPERADOR — bloqueante] Crear el fork**
   - Navegador: <https://github.com/nicobailon/pi-web-access/fork> → cuenta `ceblan` (30 s), **o** re-autenticar `gh` con classic PAT / OAuth con scope `public_repo` y repetir `gh repo fork nicobailon/pi-web-access --clone=false`.
   - Acceptance: `gh repo view ceblan/pi-web-access` devuelve el repo.

1. **Cablear `origin` y tracking**
   - File: repo `~/git-carlos/pi-extensions/pi-web-access`
   - Changes:
     ```bash
     cd ~/git-carlos/pi-extensions/pi-web-access
     git remote add origin git@github.com:ceblan/pi-web-access.git
     git fetch origin
     git branch --set-upstream-to=origin/main main
     ```
   - Acceptance: `git remote -v` muestra `origin` (ceblan) + `upstream` (nicobailon); `git status -sb` muestra `## main...origin/main`.

2. **Crear rama de trabajo**
   - Changes: `git checkout -b strip-providers` (dejar `main` limpio trackeando para merges de upstream).
   - Acceptance: `git branch --show-current` → `strip-providers`.

3. **`index.ts` — constantes EXPOSED + schema**
   - File: `index.ts` (junto a `searchProviderSchema`, ~línea 230)
   - Changes:
     ```ts
     // Strip: advertise only configured providers (runtime validation unchanged — see gemini-search.ts)
     const EXPOSED_RESOLVED_PROVIDERS = ["searxng", "tavily", "perplexity", "gemini"] as const;
     const EXPOSED_SEARCH_PROVIDERS = ["auto", "all", ...EXPOSED_RESOLVED_PROVIDERS] as const;

     function searchProviderSchema(description: string) {
     	return Type.Union([
     		StringEnum([...EXPOSED_SEARCH_PROVIDERS]),
     		Type.Array(StringEnum([...EXPOSED_RESOLVED_PROVIDERS]), { minItems: 1 }),
     	], { description });
     }
     ```
   - Si `npm run typecheck` marca `RESOLVED_SEARCH_PROVIDERS`/`SEARCH_PROVIDERS` como imports sin uso en `index.ts:10`, quitar esos dos nombres del import (conservar los `type …`).
   - Acceptance: schema generado enumera exactamente `["auto","all","searxng","tavily","perplexity","gemini"]` (scalar) y los 4 (array).

4. **`index.ts:1604-1606` — descripción de `web_search`** (string exacto, 179 tok):
   - Changes: sustituir la template literal por:
     ```
     `Search the web using SearXNG, Tavily, Perplexity, or Gemini. Pass a provider array to search those providers simultaneously, or use provider "all" to search every configured provider. Returns an AI-synthesized answer with source citations. For comprehensive research, prefer queries (plural) with 2-4 varied angles over a single query — each query gets its own synthesized answer, so varying phrasing and scope gives much broader coverage. When includeContent is true, full page content is fetched in the background. Searches auto-open the interactive browser curator and stream results live; set workflow to "none" to skip curation or "auto-summary" for a model-generated summary without the browser curator. The configured provider is used when provider is omitted or set to auto; omit provider unless explicitly overriding it. SearXNG is configured and preferred first for local/private search.`
     ```
   - Acceptance: la descripción no contiene ningún provider fuera de los 4.

5. **`index.ts:1618` — descripción del param `provider` de `web_search`**:
   - Changes: → `"Search provider or non-empty list of providers to search simultaneously; use all to search every configured provider, omit this field to use the configured provider, or use auto when none is configured"`
   - Acceptance: sin menciones a DuckDuckGo/AnySearch/xAI/Bright Data/SerpBase.

6. **`index.ts:2178` — descripción del param `provider` de `source_check`**:
   - Changes: → `"Search provider or non-empty list of providers to search simultaneously; all searches every configured provider"`
   - Acceptance: igual que task 5.

7. **Typecheck + tests**
   - Changes: `npm run typecheck && npm test` en el repo.
   - Acceptance: ambos en verde. (Los tests de runtime no tocan el schema; si alguno snapshotea descripciones, actualizar el snapshot.)

8. **Medición post-strip**
   - Changes: re-extraer los strings editados del working tree y tokenizarlos con `/tmp/tb` (script `strip-measure.mjs` ya tiene el before; hacer variante que lea del código, p.ej. `node -e` + `gpt-tokenizer` sobre el JSON del schema generado).
   - Acceptance: subtotal schema+descripciones ≤ ~400 tok (desde 819).

9. **Commit + push**
   - Changes: `git add -A && git commit -m "feat: advertise only configured providers in tool schemas (strip)" && git push -u origin strip-providers` (requiere Task 0-1).
   - Acceptance: rama visible en `github.com/ceblan/pi-web-access/tree/strip-providers`.

10. **Instalar el fork en pi**
    - File: `~/.pi/agent/npm/package.json`
    - Changes: `"pi-web-access": "git+ssh://git@github.com/ceblan/pi-web-access.git#strip-providers"` → `cd ~/.pi/agent/npm && npm install` → reiniciar pi.
    - Alternativa para probar ANTES de que exista el fork: `"pi-web-access": "file:/home/carlos/git-carlos/pi-extensions/pi-web-access"`.
    - Acceptance: `pi` arranca; las tools `web_search`/`fetch_content`/`get_search_content`/`source_check` siguen registradas.

11. **Verificación funcional (en sesión pi nueva)**
    - `/token-burden`: coste de pi-web-access ~2,050 → esperado **~1,600** (o ~1,340 con gate de source_check).
    - `web_search` sin provider → usa **searxng** (default configurado).
    - `web_search` con `provider: "all"` → ejecuta los 4 configurados.
    - `web_search` con `provider: ["tavily", "gemini"]` → array OK.
    - `fetch_content` sobre una URL cualquiera → extracción OK (firecrawl intacto; forzar una página con JS/anti-bot si se quiere ejercitar la ruta firecrawl).
    - Acceptance: todo responde sin errores de validación de enum.

12. **[Opcional, recomendado] Gate de `source_check`**
    - File: `~/.pi/web-search.json` → añadir `"tools": { "sourceCheck": { "enabled": false } }` (requiere HEAD post-gates — ya lo tenemos; NO funciona con el 0.20.0 npm).
    - Acceptance: `source_check` desaparece de la lista de tools; −262 tok adicionales. **Decisión del usuario.**

13. **[Opcional] Documentar mantenimiento del fork**
    - Añadir sección en README del fork o al final de este plan: procedimiento re-strip tras `git merge upstream/main` (los hunks son pequeños y locales; si upstream añade providers, el merge no toca las constantes `EXPOSED_*`).

14. **[Opcional] lat.md de `~/.pi`**
    - Tras ejecutar el strip: nueva sección `lat.md/extensions/` documentando el fork (por qué existe, qué stripea, cómo re-instalar). Ejecutar `lat check`.

## Files to Modify

- `~/git-carlos/pi-extensions/pi-web-access/index.ts` — constantes `EXPOSED_*` + `searchProviderSchema` (~:230), descripción `web_search` (:1604-1606), desc. param provider (:1618), desc. param provider de `source_check` (:2178), posible limpieza de imports (:10).
- `~/.pi/agent/npm/package.json` — dependencia `pi-web-access` → git URL del fork (task 10).
- `~/.pi/web-search.json` — solo si se aprueba el gate de `source_check` (task 12).

## New Files

- Ninguno en el código. (Este plan: `STRIP-PLAN.md` en la raíz del fork, sin commitear por ahora.)

## Dependencies

```
Task 0 (fork, operador) ─→ Task 1 (origin) ─→ Task 9 (push) ─→ Task 10 (install vía git)
Task 2-8 (edición local) no dependen del fork → pueden ejecutarse ya.
Task 10 (alternativa file:) permite probar todo sin fork.
Task 11 depende de 10. Task 12 depende de estar en HEAD post-gates (cumplido).
```

## Risks / Premortem

- **`auto` roto** → NO: la cadena auto (`gemini-search.ts:481+`) es hardcodeada con availability checks; no consume los enums. Además `provider:"searxng"` en config cortocircuita a searxng. Verificado leyendo `search()`.
- **`provider:"all"`** → itera `ALL_SEARCH_PROVIDERS` (16) filtrando por disponibilidad → ejecuta exactamente los 4 configurados. Funciona igual; overhead de 12 availability checks despreciable. (Con advertise-only no se toca `:93`.)
- **Provider configurado que no esté en el enum** → imposible: los 4 configurados SON los del enum. Si en el futuro se añade una key (p.ej. `exaApiKey`) sin actualizar el fork: sigue funcionando vía `auto`/default; solo no aparece anunciado. Fallo silencioso aceptable — riesgo principal del enfoque estático, mitigado por lo barato que es añadirlo (1 string).
- **El modelo pasa un provider no anunciado explícito** (p.ej. `brave` por costumbre de entrenamiento) → la tool call es **rechazada por la validación del JSON schema** (scalar y array por igual) antes de `execute()`; el modelo recibe el error de validación y se autocorrige con un valor del enum. Comportamiento deseado. Ojo al matiz gemelo: si el modelo **omite** provider o usa `auto`/`all`, el runtime puede seguir seleccionando un provider no expuesto cuando la config/disponibilidad lo determine (p.ej. si mañana se añade `exaApiKey` sin tocar el fork, `auto`/`all` lo usarán aunque no aparezca anunciado). El fallback silencioso a `"auto"` de `normalizeSearchProviderSelection` solo se ejerce en rutas **no-tool** (`/websearch`, `provider` en web-search.json, curator).
- **Merge de `upstream`** → upstream añade providers cada pocas semanas (13 en 6 semanas). Conflictos probables SOLO en la descripción de `web_search` (1 hunk); las constantes `EXPOSED_*` son código propio sin solape. Procedimiento: `git merge upstream/main`, resolver re-aplicando el string stripped, typecheck+test, push.
- **Firecrawl roto** → NO: verificado por grep; el strip no toca rutas de extracción.
- **`npm update` pisa el fork** → npm respeta el spec `git+ssh://…#branch` en package.json; no vuelve al registry mientras no se edite la línea.
- **Typecheck por imports sin uso** (`index.ts:10`) → menor; task 3 lo contempla.
- **Descripción estática miente si la config cambia** (p.ej. "SearXNG is configured…") → fork personal; aceptado. Si molesta, quitar esa última frase (+ahorro marginal).
- **`source_check` gate** → si se desactiva y luego se echa en falta, revertir es quitar 3 líneas de config.

## Todos (ejecución del strip, pendiente de aprobación)

1. [bloqueante, operador] Crear fork `ceblan/pi-web-access` (web o gh con token clásico)
2. Cablear `origin` + tracking de `main`
3. Rama `strip-providers`
4. Editar `index.ts`: constantes EXPOSED + `searchProviderSchema` + 3 descripciones (+imports si hace falta)
5. `npm run typecheck && npm test` en verde
6. Medición post-strip (≤~400 tok en los 3 elementos)
7. Commit + push `strip-providers` → origin
8. Instalar fork en `~/.pi/agent/npm` (git URL; file: como fallback pre-fork) + reiniciar pi
9. Verificación funcional: `/token-burden`, searxng default, `all`, array, fetch_content (firecrawl)
10. [opcional] Gate `source_check` en web-search.json
11. [opcional] Doc mantenimiento fork + sección lat.md en `~/.pi` + `lat check`
