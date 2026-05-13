# Path discovery heuristics

An **execution path** = one entry point + its reachable call chain.

This file gives per-language heuristics for finding entry points. The
orchestrator reads this; analyzers do not need it (they receive a path
descriptor already constructed).

## Universal scan

Before language-specific work:

```bash
# Skip these dirs always
SKIP_DIRS="node_modules .venv venv vendor target dist build .next out .cache coverage __pycache__ .git"

# Diff scope (PR mode)
git diff "$BASE"...HEAD --name-only > /tmp/changed.txt

# Component scope
# argv path is the root; descend
```

## Language-specific entry-point patterns

### TypeScript / JavaScript

HTTP routes:
- Express: `app.(get|post|put|patch|delete|use)\(`
- Fastify: `fastify.(get|post|...)\(`, `route\(`
- Koa: `router.(get|post|...)\(`
- NestJS: `@Controller`, `@Get`, `@Post`, etc.
- Next.js: files under `pages/api/`, `app/api/*/route.ts`
- tRPC: `router\(`, `procedure\.(query|mutation)`
- Hono: `app.(get|post|...)\(`

CLI / bin:
- `package.json` → `bin` field
- Files matching `bin/*.{ts,js}` or `cli.{ts,js}`
- `#!/usr/bin/env node` shebangs

Workers / queues:
- BullMQ: `Worker\(`, `Queue\(`
- AWS SDK: `.processMessage`, SQS handlers
- Pub/Sub: `.onMessage`

Exported APIs:
- `export function`, `export const X = `, `export default` at top level of
  `index.ts`, `index.js`, package roots.

Frontend (only when scope is explicitly frontend):
- React: page components, route components, error boundaries
- Vue: `pages/` directory
- Svelte: `+page.svelte`, `+server.ts`

### Python

HTTP routes:
- Flask: `@app.route`, `@bp.route`
- FastAPI: `@app.(get|post|...)`, `@router.(get|post|...)`
- Django: `urls.py` → `path(...)`, `re_path(...)`; `views.py` classes
- Starlette: `Route(...)`, `app.add_route`
- Quart: same as Flask

CLI:
- `setup.py` / `pyproject.toml` → `entry_points.console_scripts`
- `if __name__ == "__main__":` blocks
- Click / Typer: `@click.command`, `@app.command`

Workers / tasks:
- Celery: `@app.task`, `@shared_task`
- RQ: `@job`
- APScheduler: `@scheduler.scheduled_job`

Exported APIs:
- Public top-level functions/classes in `__init__.py` of packages
- Anything not prefixed with `_`

### Go

HTTP routes:
- net/http: `http.HandleFunc`, `http.Handle`, `mux.HandleFunc`
- Gin: `r.(GET|POST|...)`, `r.Handle`
- Echo: `e.(GET|POST|...)`
- Fiber: `app.(Get|Post|...)`
- Chi: `r.(Get|Post|...)`

CLI:
- `main.go` files; functions named `main()` in `package main`
- `cmd/*/main.go` (standard layout)
- Cobra: `&cobra.Command{ Run: ... }`

Workers:
- Goroutines spawned at startup (in `init()` or `main()`)
- Kafka/NSQ/RabbitMQ consumers

Exported APIs:
- Capital-letter functions/types in package roots

### Rust

HTTP routes:
- Axum: `.route("/path", get(handler))`
- Actix: `#[get("/path")]`, `App::new().service`
- Rocket: `#[get("/path")]`, `mount("/", routes![...])`
- Warp: `warp::path!("...")`

CLI:
- `main.rs`; `fn main()`
- Clap: `#[derive(Parser)]`, `Command::new`

Exported APIs:
- `pub fn`, `pub struct`, `pub enum` at crate / module roots

### Java / Kotlin

HTTP routes:
- Spring: `@RestController`, `@RequestMapping`, `@GetMapping`, etc.
- Jakarta REST: `@Path`, `@GET`, etc.
- Ktor: `routing { get("...") { } }`
- Micronaut: `@Controller`, `@Get`

CLI:
- `public static void main(String[] args)`
- Picocli: `@Command`

Background:
- `@Scheduled`, `@KafkaListener`, `@RabbitListener`, `@EventListener`

### Ruby

HTTP routes:
- Rails: `config/routes.rb` → `get '/path' => 'controller#action'`
- Sinatra: `get '/path' do ... end`

CLI:
- `bin/*` scripts, files with `#!/usr/bin/env ruby`

Background:
- Sidekiq workers: `include Sidekiq::Worker`
- Resque: `extend Resque::Job`

### PHP

HTTP routes:
- Laravel: `Route::get('/path', ...)`, controller methods
- Symfony: `#[Route('/path')]`
- Slim: `$app->get('/path', ...)`

CLI:
- Laravel Artisan: `php artisan` commands (`app/Console/Commands/`)
- Symfony: `Console\Command` classes

### Swift

iOS apps: View controllers; `@main` annotated structs; SwiftUI `App` conformance.

### C# / .NET

HTTP routes:
- ASP.NET: `[Route]`, `[HttpGet]`, `MapGet`, `MapPost`
- Minimal API: `app.MapGet`, `app.MapPost`

CLI:
- `static void Main`, `static async Task Main`

## Building the reachable call chain

For a discovered entry point at `<file>:<symbol>`:

1. Read the file. Identify direct callees in `<symbol>` (function calls).
2. Resolve each callee to its defining file via:
   - Import statements at the top of `<file>`
   - Project's module resolution rules
3. Recurse, bounded by:
   - Max depth: 4 levels
   - Skip standard library and third-party packages (anything in node_modules/,
     vendor/, .venv/, etc.)
   - Skip if you've already added the file
4. Collect the union of files; this is `reachable_files`.

If the language has language-server tooling available locally (TypeScript's
`tsserver`, Go's `gopls`, Rust's `rust-analyzer`), you may shell out to query
callee/caller relationships. Otherwise, Grep with the symbol name across the
repo gives a reasonable approximation.

## Intersection with diff (PR mode)

After building `reachable_files` for each entry point:

```
diff_intersection = reachable_files ∩ changed_files
```

If `diff_intersection` is empty, **drop the path** — the PR doesn't touch it.

If `diff_intersection` is non-empty, **keep the path** and include it in the
descriptor. The analyzer will pay special attention to those files.

## Caps and limits

- Max paths per run: 50 (default). Above this, ask user.
- Max reachable_files per path: 30. Above this, truncate to the most relevant
  30 (those in `diff_intersection` first, then by closeness to the entry).
- Max depth from entry: 4.

## Edge cases

- **Multiple entry-point styles for the same handler** (e.g., a Flask route
  also called from a CLI command): create two paths, one per entry. They will
  be deduped if their candidates collide.
- **Generic libraries (no obvious entry point):** treat every exported public
  function as an entry point.
- **Monorepo:** scope discovery to the package containing the diff. Don't walk
  into sibling packages unless the diff touches them.
