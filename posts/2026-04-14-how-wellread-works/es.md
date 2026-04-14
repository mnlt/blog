# Cómo funciona wellread por dentro

En el [post anterior](../2026-04-13-wellread/es.md) conté por qué construí wellread. Aquí cuento cómo funciona.

## El flujo

Cuando le preguntas algo a Claude con wellread instalado, pasan varias cosas antes de que busque en la web:

1. Un hook (UserPromptSubmit) intercepta tu mensaje antes de que Claude lo procese
2. El hook inyecta una instrucción de 4 líneas: "busca en wellread primero, guarda después"
3. El hook también lee tu JSONL local para saber tu baseline actual de contexto
4. Claude sanitiza tu pregunta — quita nombres de proyecto, rutas, credenciales — y llama a wellread
5. Wellread busca con embeddings + texto completo
6. Dependiendo del resultado: hit, partial o miss

### Hit

Wellread devuelve un resumen de ~600 tokens. Claude responde con eso. No toca la web. Un solo turno.

### Partial

Wellread devuelve lo que tiene. Claude lee los gaps (lo que no se exploró) y solo investiga eso. Cuando termina, guarda la versión actualizada.

### Miss

Claude investiga normal — web searches, fetches, razonamiento. Cuando termina, guarda el resultado en wellread para el siguiente.

## La búsqueda

La query que llega a wellread no es lo que tú escribiste. Claude la transforma en un concepto técnico genérico. Si preguntas "cómo configuro auth en mi proyecto de Next.js que está en /Users/mnlt/projects/miapp", lo que llega a wellread es algo como "how to set up authentication in Next.js App Router with Auth.js".

Nada privado sale de tu máquina que no sea la pregunta genérica.

La búsqueda combina dos cosas:
- **Embeddings** (OpenAI, text-embedding-3-small): busca por significado, no por palabras exactas. "configurar autenticación" matchea con "authentication setup"
- **Texto completo** (BM25): busca por palabras clave. Captura lo que los embeddings se pierden — nombres de librerías, versiones específicas

Los resultados se fusionan con Reciprocal Rank Fusion (70% semántico, 30% texto). Devuelve hasta 5 resultados.

## Freshness

No todo el conocimiento caduca igual. Los fundamentos de TCP no cambian. Una beta de una librería puede cambiar mañana.

Cada research se guarda con un nivel de volatilidad:

| Nivel | Ejemplo | Fresco | Check | Stale |
|---|---|---|---|---|
| Timeless | TCP, SQL basics | 1 año | — | después |
| Stable | React, PostgreSQL | 6 meses | 1 año | después |
| Evolving | Next.js, Bun | 30 días | 90 días | después |
| Volatile | betas, pre-releases | 7 días | 30 días | después |

El agente que contribuye el research es quien decide la volatilidad. No es perfecto, pero funciona razonablemente bien.

Cuando alguien hace hit de una entry en estado "check", Claude no se fía — hace un web search rápido para verificar que sigue vigente. Si confirma, resetea el reloj para el siguiente. Si encuentra cambios, lo reemplaza con una versión nueva.

Así el conocimiento se mantiene fresco sin que nadie tenga que hacer mantenimiento manual.

## Gaps

Cada research se guarda con una lista de "lo que no se exploró". Si alguien buscó "auth en Next.js" pero no cubrió OAuth con Google, eso queda como gap explícito.

Cuando otro agente llega con una pregunta que el research cubre parcialmente, sabe exactamente qué investigar y qué no. No repite lo que ya está, no se inventa lo que falta — busca solo los gaps.

## Replaces y versiones

Si investigas algo que ya existía pero con info nueva, tu research reemplaza al anterior. El anterior deja de aparecer en búsquedas pero no se borra — se mantiene la cadena de versiones.

Esto es importante porque las fuentes y los tokens ahorrados se acumulan entre versiones. Un research que empezó con 3 fuentes y ha sido reemplazado 2 veces puede tener 8 fuentes y un historial de cómo el conocimiento evolucionó.

## Cómo se miden los tokens

Lo más difícil de resolver. Necesitaba saber cuántos tokens ahorra realmente un hit de wellread. No una estimación — el dato real.

### PostToolUse hook

Después de cada save, un hook (PostToolUse) se dispara automáticamente. Este hook:

1. Lee el JSONL de tu sesión actual
2. Encuentra el span entre el wellread search y el wellread save
3. Suma el `input_tokens + cache_creation + cache_read` de cada turn en ese span
4. Resta el baseline (lo que se habría enviado de todas formas)
5. Envía el resultado al servidor via `PATCH /measure`

El resultado es el coste incremental real del research — no una estimación, el dato exacto del JSONL.

### El badge personalizado

Cuando haces hit, el badge no muestra cuánto costó el research original. Muestra cuánto te habría costado **a ti**, con **tu** baseline actual.

La fórmula: `raw_tokens + (tu_baseline × research_turns) - response_tokens`

Un research que costó 200K en una sesión nueva, te puede costar 3.5M si estás en el turn 100. El badge te muestra ese 3.5M.

Para esto el UserPromptSubmit hook lee tu JSONL y pasa tu baseline actual como `client_stats`. El servidor hace el cálculo con los datos del research (raw_tokens, research_turns) y tu baseline.

## El hook

El hook es lo que hace que wellread funcione de verdad en Claude Code. Sin el hook, el agente tendría que decidir por sí mismo si buscar en wellread. Con el hook, busca siempre.

Es un bash script de 15 líneas que se instala en `~/.wellread/hook.sh`. Lo que hace:

1. Lee tu mensaje (si es muy corto, lo ignora — no busca para "hey" o "ok")
2. Lee tu baseline del JSONL (última línea con usage)
3. Inyecta 4 líneas de instrucciones al system prompt

Las instrucciones son deliberadamente mínimas. Cada token del hook se re-envía en cada turn de la sesión — un hook de 500 tokens cuesta 500 tokens × N turns. Wellread son ~106 tokens.

Lo otro: todo lo posible vive en el servidor. Si cambia la lógica de búsqueda, freshness o badges, todos los usuarios lo reciben sin hacer nada. Solo si cambio el hook necesitan hacer `npx wellread@latest`.

## El instalador

`npx wellread` detecta qué herramientas tienes instaladas y configura todo:

- **Claude Code**: MCP server en settings.json + hook UserPromptSubmit + hook PostToolUse (para medición)
- **Cursor**: MCP server en mcp.json + regla en rules
- **Windsurf, Gemini CLI, VS Code, OpenCode**: equivalentes para cada uno

Sin API keys manuales. El instalador registra un usuario anónimo y guarda la key en la config del cliente.

Si quieres probar: `npx wellread`

GitHub: https://github.com/mnlt/wellread
