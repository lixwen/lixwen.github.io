# Project Instructions: 油菜地 (Hexo Blog)

This is a Hexo-based personal blog project.

## Architecture & Framework
- **Framework:** Hexo 7.3.0
- **Theme:** Cactus (located in `themes/cactus`)
- **Primary Language:** Chinese (zh-CN)
- **Timezone:** Asia/Shanghai

## Key Workflows
- **Development Server:** `npm run server` (starts local server at http://localhost:4000)
- **New Post:** `hexo new post "Title"` (creates a new markdown file in `source/_posts`)
- **Build Site:** `npm run build` (generates static files in `public/`)
- **Clean:** `npm run clean` (clears the cache and generated files)
- **Deploy:** `npm run deploy` (deploys to the configured destination)

## Content Structure
- `source/_posts/`: All blog posts are stored here as Markdown files.
- `source/images/`: (Standard practice) Store post-related images here.
- `_config.yml`: Main site configuration.
- `themes/cactus/_config.yml`: Theme-specific configuration.

## Conventions
- **Post Naming:** Follow `:title.md` pattern.
- **Language:** Posts should generally be written in Chinese as per site configuration.
- **Front-matter:** Ensure every post has `title`, `date`, and `tags` (if applicable).
