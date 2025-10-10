# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Mintlify documentation site. Mintlify is a documentation platform that uses MDX files (Markdown with JSX components) to create interactive documentation. The repository contains documentation content, configuration, and assets for a docs site.

## Development Commands

**Local development:**
```bash
mint dev
```
Starts a local preview server at `http://localhost:3000`. Must be run from the directory containing `docs.json`.

**Custom port:**
```bash
mint dev --port 3333
```

**Update CLI:**
```bash
npm mint update
```
Run this if local preview doesn't match production version.

**Validate links:**
```bash
mint broken-links
```
Checks for broken links in the documentation.

**Install CLI:**
```bash
npm i -g mint
```

## Architecture

### Configuration
- `docs.json`: Central configuration file defining the entire site structure, including:
  - Theme and branding (colors, logo, favicon)
  - Navigation structure with tabs and groups
  - Contextual menu options (copy, view, chatgpt, claude, perplexity, mcp, cursor, vscode)
  - External links (navbar, footer, global anchors)

### Content Structure
- **MDX files**: All documentation content is written in MDX (`.mdx` extension)
- **File-based routing**: Each MDX file corresponds to a page; file paths determine URLs
- **Navigation mapping**: Files must be registered in `docs.json` navigation structure to appear in the site
- **Content organization**:
  - Root level: Main pages (index, quickstart, development)
  - `essentials/`: Core documentation features (markdown, navigation, settings, code, images, reusable-snippets)
  - `ai-tools/`: AI tool integration guides (cursor, claude-code, windsurf)
  - `api-reference/`: API documentation (introduction, endpoint examples)
  - `snippets/`: Reusable MDX snippets that can be included in multiple pages

### Assets
- `logo/`: Light and dark theme logos (SVG)
- `images/`: Documentation images and screenshots
- `favicon.svg`: Site favicon
- `api-reference/openapi.json`: OpenAPI specification for auto-generated API docs

## Mintlify-Specific Components

When editing MDX files, you can use Mintlify's built-in components:
- `<Card>`: Clickable card with title, icon, and link
- `<Columns>`: Multi-column layout
- `<Steps>` and `<Step>`: Step-by-step guides
- `<Info>`, `<Warning>`, `<Tip>`: Callout boxes
- `<AccordionGroup>` and `<Accordion>`: Collapsible sections
- `<Frame>`: Image wrapper with styling

## Prerequisites

- Node.js version 19 or higher
- Mintlify CLI (`mint`) installed globally

## Deployment

Changes pushed to the main branch are automatically deployed to production via GitHub app integration (configured in Mintlify dashboard at dashboard.mintlify.com/settings/organization/github-app).


# Mintlify documentation

## Working relationship
- You can push back on ideas-this can lead to better documentation. Cite sources and explain your reasoning when you do so
- ALWAYS ask for clarification rather than making assumptions
- NEVER lie, guess, or make up information

## Project context
- Format: MDX files with YAML frontmatter
- Config: docs.json for navigation, theme, settings
- Components: Mintlify components

## Content strategy
- Document just enough for user success - not too much, not too little
- Prioritize accuracy and usability of information
- Make content evergreen when possible
- Search for existing information before adding new content. Avoid duplication unless it is done for a strategic reason
- Check existing patterns for consistency
- Start by making the smallest reasonable changes

## Frontmatter requirements for pages
- title: Clear, descriptive page title
- description: Concise summary for SEO/navigation

## Writing standards
- Second-person voice ("you")
- Prerequisites at start of procedural content
- Test all code examples before publishing
- Match style and formatting of existing pages
- Include both basic and advanced use cases
- Language tags on all code blocks
- Alt text on all images
- Relative paths for internal links

## Git workflow
- NEVER use --no-verify when committing
- Ask how to handle uncommitted changes before starting
- Create a new branch when no clear branch exists for changes
- Commit frequently throughout development
- NEVER skip or disable pre-commit hooks

## Do not
- Skip frontmatter on any MDX file
- Use absolute URLs for internal links
- Include untested code examples
- Make assumptions - always ask for clarification