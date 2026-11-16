# CLAUDE.md

## Qué es

«Nuestro pacto»: una página web para que una pareja (Valukis = persona `a`, Tomilin = persona `b`) siga hábitos juntos. Cada racha activa suma «días de progreso» que desbloquean regalos que el otro debe hacer (7 días, 15, 1 mes, 3, 6, 1 año y luego cada año). Si alguien abandona un hábito, paga un castigo (regalo o ruleta). Las reglas de negocio están explicadas en detalle en [README.md](README.md).

Toda la interfaz y los textos están en **español de Chile** (`es-CL`), en tono cercano. Mantén ese idioma y tono en la UI, los comentarios y los mensajes de commit.

## Principios del proyecto (no negociables)

1. **Solo dos usuarios.** La usamos él y su pareja. No diseñes para escalabilidad, multiusuario, roles, rate limiting, auditoría ni seguridad robusta. Un PIN opcional es todo lo que hace falta. No agregues validaciones defensivas ni capas de abstracción «por si acaso».
2. **Simple.** Sin frameworks, sin bundler, sin paso de compilación, sin dependencias npm. HTML + CSS + JavaScript plano. Si algo se puede resolver con menos código, mejor. No agregues servicios nuevos; los que ya hay (Vercel + Supabase/Upstash) funcionan y se quedan.

## Despliegue y datos

- Se aloja en **Vercel**. [api/state.js](api/state.js) es la única función serverless: guarda todo el estado en **Supabase** (tabla `pacto`, ver [supabase.sql](supabase.sql)) o en **Upstash Redis**, según cuál esté conectado (prioriza Upstash si ambos existen).
- En Supabase hay una sola fila (`id = 'main'`) con el estado completo como JSON en `state` y un contador `version`. **Cambiar la forma de los datos no requiere tocar la tabla**: se hace en `defaultState()` / `migrate()` de [app.js](app.js). Los cambios a la tabla en sí se aplican a mano en el SQL Editor de Supabase; editar [supabase.sql](supabase.sql) aquí no los aplica.
- Contrato de la API (no lo rompas sin actualizar ambos lados):
  - `GET /api/state` → `{ state, version }`.
  - `PUT /api/state` con `{ state, baseVersion }` → `{ version }`, o `409 { state, version }` si otro teléfono guardó antes.
  - Header opcional `x-pin`, comparado con la variable de entorno `APP_PIN` → `401` si no coincide.
- `respaldos/` (ignorada por git) guarda respaldos descargados desde Ajustes. Contienen datos reales: úsalos para probar cambios de lógica, nunca los subas.

## Arquitectura del cliente ([app.js](app.js))

Un solo archivo, sin módulos. Secciones en orden:

- **Utilidades**: fechas como strings `YYYY-MM-DD` en hora local (`ymd`, `parse`, `addDays`, `diffDays`). Compara fechas como strings; no uses `Date` con UTC.
- **Estado**: todo vive en un único objeto JSON (`defaultState()`): `title`, `people.{a,b}`, `settings`, `habits[]`, `rewards{}`, `debts[]`, `log[]`. `migrate()` rellena campos faltantes; si agregas un campo nuevo al estado, agrégalo también ahí para que los datos viejos sigan funcionando.
- **Reglas**:
  - `timeline(h)` recorre día a día un hábito. **La racha (`streak`) es la cantidad de días marcados** desde que empezó la racha actual. Un día marcado suma 1 y reinicia los fallos. Un día libre (`restDays`) no suma, no es fallo y también reinicia los fallos. Cada día sin marcar suma un fallo, y pasar de `maxMiss` corta la racha. **Hoy nunca corta la racha** y solo suma si está marcado.
  - `progress(h)` es el camino de regalos de **un** hábito: cada día marcado suma 1. Si la racha se corta, los puntos vuelven al último hito alcanzado (`base`). Solo generan regalos los hábitos con `rewards !== false` que no fueron abandonados (`hasRewards(h)`).
  - `milestone(i)` devuelve el hito i; después del último se repite cada `repeatEvery` días.
  - Los regalos se identifican con la clave `"<idHábito>:<días>"` en `state.rewards` y guardan `habitId`, `habitName` y `person`. Nunca se borran. `migrate()` descarta los regalos antiguos sin `habitId` (de cuando el progreso era por persona).
  - `h.rewards` se elige al crear el hábito y no se puede cambiar. Los hábitos sin premios no tienen camino ni regalo final, pero sí castigo al rendirse. El castigo «regalo del nivel» usa días de calendario desde `h.start`.
- **Sincronización**: `mutate(fn)` aplica el cambio sobre una copia, renderiza y hace `PUT`; ante un `409` recarga el estado del servidor y vuelve a aplicar `fn` (hasta 5 intentos). Toda modificación del estado debe pasar por `mutate` y `fn` debe ser re-aplicable sobre un estado distinto (busca por `id`, no guardes referencias). `refresh()` consulta cada 15 s.
- **Render**: funciones `renderX()` que generan HTML con template strings. Escapa siempre el texto del usuario con `esc()`.
- **Acciones**: delegación de eventos con `data-action` (objeto `actions`) y formularios con `data-form` (objeto `forms`). Para un botón nuevo, agrega una entrada ahí en vez de listeners sueltos.
- **localStorage** (por teléfono, no compartido): `pacto:me` (quién es este teléfono), `pacto:pin`, `pacto:seen` (regalos ya celebrados).

Solo el dueño de un hábito puede marcarlo o terminarlo (`h.owner === me`).

## Estilos ([styles.css](styles.css))

- Colores como variables en `:root`, redefinidas en `@media (prefers-color-scheme: dark)`. Cualquier color nuevo debe tener su versión oscura.
- `.p-a` / `.p-b` definen `--pc` y `--pc-soft` (rosado para `a`, verde azulado para `b`); úsalos para colorear por persona.
- Pensado para celular primero (se usa como app agregada a la pantalla de inicio). Respeta `prefers-reduced-motion`.

## Cómo probar

No hay tests ni build. La página no funciona abierta como archivo (`file://`): necesita la API, así que se prueba en un deploy de Vercel (o con `vercel dev` y las variables de entorno de la base de datos). Para simular a los dos usuarios, usa dos ventanas (una normal y una de incógnito) y elige un perfil distinto en cada una. Las reglas de rachas y progreso se pueden verificar a mano llamando `timeline()` y `progress()` desde la consola del navegador.
