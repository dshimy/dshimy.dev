# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal blog for Darian Shimy at dshimy.dev. Built with Astro 6, MDX, and deployed to GitHub Pages via GitHub Actions on push to `main`.

## Commands

- `npm run dev` — Start dev server
- `npm run build` — Production build (outputs to `dist/`)
- `npm run preview` — Preview production build locally

Requires Node >= 22.12.0.

## Architecture

- **Content**: Blog posts live in `src/content/blog/` as Markdown/MDX files. Frontmatter schema (title, description, pubDate, optional updatedDate, optional heroImage) is enforced in `src/content.config.ts`.
- **Pages**: `src/pages/blog/[...slug].astro` generates static pages from the blog collection. `src/pages/rss.xml.js` generates the RSS feed.
- **Layout**: Single layout `src/layouts/BlogPost.astro` wraps all posts. Shared components (Header, Footer, BaseHead, FormattedDate) in `src/components/`.
- **Constants**: Site title and description are defined in `src/consts.ts` and imported where needed.
- **Styling**: Global CSS in `src/styles/global.css` with component-scoped `<style>` blocks in `.astro` files. Uses CSS custom properties (e.g., `--gray`, `--accent`, `--box-shadow`).
- **Deployment**: Push to `main` triggers `.github/workflows/deploy.yml` which builds and deploys to GitHub Pages.
