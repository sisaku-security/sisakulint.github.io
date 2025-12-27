# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the documentation website for **sisakulint**, a static analysis security tool (SAST) for GitHub Actions workflows. The site is built using Hugo and the hugo-book theme, and is deployed to GitHub Pages.

**What is sisakulint?** A CI-friendly static linter with SAST semantic analysis for GitHub Actions that detects security vulnerabilities in workflow files (e.g., credential leaks, injection attacks, permission issues, deprecated commands).

## Development Commands

### Initial Setup

```bash
python -m venv .venv
source .venv/bin/activate  # On Linux/Mac
# OR
.venv\Scripts\activate  # On Windows
pip install -r requirements.txt
```

### Running the Development Server

```bash
# Start the OGP proxy server (required for link previews)
python ogp_proxy.py &

# Start Hugo server
hugo server

# With cache invalidation (useful when debugging)
hugo server --disableFastRender --ignoreCache
```

The site will be available at `http://localhost:1313/`

### Building for Production

```bash
hugo --minify
```

Output goes to `./public` directory.

## Architecture

### Hugo + Python Hybrid

This is a Hugo static site with a Python Flask service for fetching Open Graph Protocol (OGP) metadata:

- **Hugo**: Static site generator using the `hugo-book` theme module
- **OGP Proxy** (`ogp_proxy.py`): Flask server running on port 3000 that fetches OGP metadata for link previews
- **Custom Shortcode** (`layouts/shortcodes/popup_link2.html`): Hugo shortcode that calls the OGP proxy to render rich link previews

### Content Structure

```
content/
├── _index.md              # Homepage with tool overview and usage examples
└── docs/
    ├── author/            # Author profiles (fujioka.md, sada.md)
    └── rules/             # Documentation for each security rule
        ├── CredentialsRule.md
        ├── commitSHARule.md
        ├── idRule.md
        ├── permissions.md
        ├── timeoutminutesrule.md
        └── workflowcall.md
```

### Theme Configuration

The site uses the `hugo-book` theme (imported as a Hugo module via `go.mod`). The theme is not vendored locally - Hugo fetches it from GitHub.

### OGP Proxy Dependency

**Important**: The OGP proxy server MUST be running during development for the `popup_link2` shortcode to work. This shortcode is used extensively on the homepage to render GitHub repository links with rich previews. If the proxy isn't running:
- Hugo builds will hang or fail when calling `http://localhost:3000/ogp`
- Link preview cards won't render properly

The proxy fetches OGP metadata (title, description, image, favicon) from external URLs and provides fallbacks using Google's favicon service if fetching fails.

## Adding New Rule Documentation

When documenting a new sisakulint rule:

1. Create a new markdown file in `content/docs/rules/`
2. Add frontmatter with title and weight (for ordering)
3. Reference the GitHub Actions documentation for the relevant security feature
4. Include examples of what the rule detects and why it matters

Example structure:
```markdown
---
title: "Rule Name"
weight: N
---

## Overview
Brief description of what this rule checks

## Why This Matters
Security implications of violations

## Examples
Example workflow snippets showing violations
```

## Deployment

The site is automatically deployed via GitHub Actions (`.github/workflows/*.yml`) when changes are pushed to the `main` branch. The workflow:

1. Sets up Python environment and installs dependencies
2. Starts the OGP proxy server in the background
3. Runs `hugo --minify` to build the site
4. Deploys to GitHub Pages using `actions/deploy-pages`

## Module Management

This project uses Go modules for Hugo theme dependencies. To update the hugo-book theme:

```bash
hugo mod get -u github.com/alex-shpak/hugo-book
hugo mod tidy
```
