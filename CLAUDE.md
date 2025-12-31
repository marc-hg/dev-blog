# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About Quartz

Quartz is a static site generator for publishing digital gardens and notes as websites. It transforms Markdown files into a fully-featured website with features like backlinks, graph visualization, search, and more.

## Development Commands

### Building and Testing
```bash
# Build the site (outputs to public/)
npx quartz build

# Build with dev server (hot reload on http://localhost:8080)
npx quartz build --serve

# Build with file watching (rebuild on changes)
npx quartz build --watch

# Type check without emitting
npm run check

# Format code
npm run format

# Run tests
npm test
```

### CLI Options for `quartz build`
- `-d, --directory <path>` - Content folder (default: "content")
- `-o, --output <path>` - Output folder (default: "public")
- `--serve` - Run local development server
- `--port <number>` - HTTP server port
- `--wsPort <number>` - WebSocket port for hot reload
- `--concurrency <number>` - Worker thread count (auto-detected by default)
- `--verbose` - Enable debug logging

### Other CLI Commands
```bash
npx quartz create     # Initialize Quartz in a directory
npx quartz sync       # Git sync with upstream
npx quartz update     # Pull Quartz framework updates
npx quartz restore    # Recover content from cache
```

## Architecture Overview

### Three-Stage Build Pipeline

Quartz processes content through three distinct stages:

**1. Parse** (`quartz/processors/parse.ts`)
- Reads Markdown files and transforms them into HTML AST (HAST)
- Uses worker threads for parallel processing (chunks of 128 files)
- Pipeline: Raw Text → textTransform → MDAST (via remark) → markdownPlugins → HAST (via remark-rehype) → htmlPlugins
- Output: `ProcessedContent[]` (tuples of [HAST tree, VFile metadata])

**2. Filter** (`quartz/processors/filter.ts`)
- Applies filter plugins sequentially to determine what content to publish
- Each filter's `shouldPublish()` method returns boolean
- Examples: RemoveDrafts, ExplicitPublish

**3. Emit** (`quartz/processors/emit.ts`)
- Runs all emitter plugins in parallel to generate output files
- Each emitter receives filtered content and writes HTML/JSON/RSS/etc
- Supports incremental builds via `partialEmit()` method

### Plugin System

Quartz has three plugin types, each serving a different stage:

**Transformers** (`QuartzTransformerPlugin`)
- Modify content during the parse stage
- Three hooks:
  - `textTransform(ctx, src)` - Preprocess raw text
  - `markdownPlugins(ctx)` - Return remark plugins for MDAST manipulation
  - `htmlPlugins(ctx)` - Return rehype plugins for HAST manipulation
- Can provide external resources (CSS/JS) via `externalResources()`
- Examples: FrontMatter, GitHubFlavoredMarkdown, SyntaxHighlighting, Latex, CrawlLinks
- Location: `quartz/plugins/transformers/`

**Filters** (`QuartzFilterPlugin`)
- Determine which content gets published
- Single method: `shouldPublish(ctx, content) → boolean`
- Run sequentially during filter stage
- Location: `quartz/plugins/filters/`

**Emitters** (`QuartzEmitterPlugin`)
- Generate output files from processed content
- Main method: `emit(ctx, content, resources) → Promise<FilePath[]>`
- Optional `partialEmit()` for incremental builds
- Run in parallel during emit stage
- Examples: ContentPage, TagPage, FolderPage, ContentIndex, Assets, Static
- Location: `quartz/plugins/emitters/`

### Plugin Factory Pattern

All plugins use a factory pattern that returns a plugin instance:

```typescript
export const MyPlugin: QuartzTransformerPlugin<Options> = (userOpts) => ({
  name: "MyPlugin",
  markdownPlugins(ctx) {
    return [/* remark plugins */]
  },
  htmlPlugins(ctx) {
    return [/* rehype plugins */]
  },
})
```

### Key Data Structures

**VFile** - Central data carrier throughout the pipeline:
- `vfile.data.slug` - URL slug for the page
- `vfile.data.filePath` - Original file path
- `vfile.data.frontmatter` - Parsed YAML/TOML metadata
- `vfile.data.aliases` - Alternate slugs for redirects
- `vfile.data.dates` - Created/modified/published timestamps

**BuildCtx** - Context object passed to all plugins:
- `ctx.cfg` - Quartz configuration from `quartz.config.ts`
- `ctx.argv` - CLI arguments
- `ctx.allSlugs` - All page slugs (for link resolution)
- `ctx.buildId` - Unique build identifier

**ProcessedContent** - Tuple of `[HAST, VFile]` representing parsed content

### Component System

Components are Preact UI elements that render parts of the page:
- Location: `quartz/components/`
- Configured in `quartz.layout.ts` (not `quartz.config.ts`)
- Three layout sections:
  - `sharedPageComponents` - Head, header, footer (all pages)
  - `defaultContentPageLayout` - Article pages (breadcrumbs, TOC, sidebar)
  - `defaultListPageLayout` - Tag/folder index pages
- Receive `QuartzComponentProps` with page data and config
- Examples: ArticleTitle, Breadcrumbs, Explorer, Graph, Search, TableOfContents

### Directory Structure

```
quartz/
├── bootstrap-cli.mjs       # CLI entry point (yargs-based)
├── build.ts                # Main build orchestration
├── cfg.ts                  # Configuration type definitions
├── cli/                    # Command handlers (create, build, sync, etc)
├── components/             # Preact UI components
│   ├── scripts/            # Client-side JavaScript
│   ├── styles/             # Component-specific styles
│   └── renderPage.tsx      # Page rendering logic
├── i18n/                   # Internationalization (30+ locales)
├── plugins/                # Plugin implementations
│   ├── transformers/       # Content transformation plugins
│   ├── filters/            # Content filtering plugins
│   └── emitters/           # Output generation plugins
├── processors/             # Core pipeline stages (parse, filter, emit)
├── static/                 # Static assets bundled with output
├── styles/                 # Global stylesheets
├── util/                   # Utility functions
└── worker.ts               # Worker thread for parallel parsing

Root:
├── quartz.config.ts        # Site configuration and plugin setup
├── quartz.layout.ts        # UI component layout
└── content/                # User's Markdown content (default)
```

## Configuration

### quartz.config.ts

Primary configuration file for the entire site:

- `configuration.pageTitle` - Site title
- `configuration.enableSPA` - Single-page app mode (prevents FOUC)
- `configuration.enablePopovers` - Link preview popups
- `configuration.theme` - Colors, typography, fonts
- `configuration.baseUrl` - For CNAME, sitemaps, RSS
- `configuration.ignorePatterns` - Glob patterns to exclude (e.g., ["private", ".obsidian"])
- `configuration.defaultDateType` - How to determine page dates ("modified" | "created" | "published")
- `configuration.locale` - Language for i18n (e.g., "en-US")
- `configuration.analytics` - Analytics provider config (Plausible, Google, Umami, etc.)
- `plugins.transformers` - Array of transformer plugins
- `plugins.filters` - Array of filter plugins
- `plugins.emitters` - Array of emitter plugins

### quartz.layout.ts

UI component layout configuration:

- `sharedPageComponents.head` - Head component (all pages)
- `sharedPageComponents.header` - Header component (all pages)
- `sharedPageComponents.afterBody` - Footer component (all pages)
- `defaultContentPageLayout.beforeBody` - Components before content (e.g., Breadcrumbs, ArticleTitle)
- `defaultContentPageLayout.left` - Left sidebar components (e.g., PageTitle, Search, Explorer)
- `defaultContentPageLayout.right` - Right sidebar components (e.g., Graph, TableOfContents, Backlinks)
- `defaultListPageLayout` - Similar structure for tag/folder pages

## Key Technologies

- **Build System**: esbuild for bundling, workerpool for parallelization
- **Markdown Processing**: unified, remark (MDAST), rehype (HAST)
- **UI Framework**: Preact (lightweight React alternative)
- **Styling**: Sass via esbuild-sass-plugin, LightningCSS for optimization
- **File Watching**: chokidar for incremental builds
- **CLI**: yargs for command parsing
- **Search**: FlexSearch for client-side search index
- **Graph**: D3.js for interactive graph visualization
- **Git**: @napi-rs/simple-git for date extraction

## Working with Plugins

### Creating a Transformer

1. Create file in `quartz/plugins/transformers/`
2. Export a factory function that returns a `QuartzTransformerPlugin`
3. Implement transformation hooks (textTransform, markdownPlugins, or htmlPlugins)
4. Add to `quartz.config.ts` plugins.transformers array
5. Export from `quartz/plugins/index.ts` for user access

Example:
```typescript
export const MyTransformer: QuartzTransformerPlugin = () => ({
  name: "MyTransformer",
  markdownPlugins() {
    return [myRemarkPlugin]
  },
})
```

### Creating an Emitter

1. Create file in `quartz/plugins/emitters/`
2. Export a factory function that returns a `QuartzEmitterPlugin`
3. Implement `emit()` method to generate files
4. Optionally implement `partialEmit()` for incremental builds
5. Add to `quartz.config.ts` plugins.emitters array

### Creating a Component

1. Create file in `quartz/components/`
2. Export a Preact component function that takes `QuartzComponentProps`
3. Add static properties: `css` (optional styling), `beforeDOMLoaded` (optional client script)
4. Add to `quartz.layout.ts` in appropriate layout section

## Incremental Builds

Quartz supports incremental builds when running with `--watch`:

- `contentMap` tracks all files (markdown + static)
- `changesSinceLastBuild` records file changes since last build
- Emitters can implement `partialEmit()` to optimize based on changes
- Build locking via `Mutex` prevents concurrent builds
- Each build has a unique `buildId` to prevent race conditions

## Testing

Tests use Node's built-in test runner:
```bash
npm test                    # Run all tests
tsx --test path/to/test.ts  # Run specific test file
```

Test files are typically colocated with source files and named `*.test.ts`.

## Important Patterns

### Unified/Remark/Rehype Pipeline

Content transformation uses the unified ecosystem:
- **remark-parse** - Markdown → MDAST
- **remark plugins** - Transform MDAST (e.g., remark-gfm, remark-math)
- **remark-rehype** - MDAST → HAST
- **rehype plugins** - Transform HAST (e.g., rehype-katex, rehype-slug)
- **hast-util-to-html** - HAST → HTML string

Plugins can inject their own remark/rehype plugins via `markdownPlugins()` and `htmlPlugins()`.

### Context Injection

`BuildCtx` is passed to all plugins, allowing coordination:
- Plugins can access global config via `ctx.cfg`
- Link resolution uses `ctx.allSlugs` to validate internal links
- Shared resources optimized across all emitters

### Worker Thread Pooling

Markdown parsing uses worker threads for performance:
- Spawns workers via `workerpool` (default: CPU count)
- Chunks files into groups of 128
- Each worker runs in isolated process with own TypeScript context
- Worker implementation in `quartz/worker.ts`, bootstrapped by `bootstrap-worker.mjs`

## Performance Considerations

- **Disable CustomOgImages** during development (comment out in `quartz.config.ts`) - it significantly slows builds
- Use `--concurrency=1` flag to profile single-threaded performance
- Incremental builds (`--watch`) are much faster than full rebuilds
- Large content vaults benefit from worker thread parallelization
- SPA mode (`enableSPA: true`) improves perceived performance by preventing page reloads
