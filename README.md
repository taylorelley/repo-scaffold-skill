# repo-scaffold

[![skills.sh](https://skills.sh/b/taylorelley/repo-scaffold-skill)](https://skills.sh/taylorelley/repo-scaffold-skill)

A Claude Code skill that scaffolds a complete, production-ready React + PocketBase project from an empty directory.

## Install

```bash
npx skills add taylorelley/repo-scaffold-skill
```

Or install globally:

```bash
npx skills add taylorelley/repo-scaffold-skill -g
```

## Stack

| Layer | Choice |
|---|---|
| Framework | React 19 + TypeScript |
| Bundler | Vite (latest) |
| Routing | TanStack Router (file-based) |
| Data fetching | TanStack Query v5 |
| Styling | Tailwind CSS v4 |
| Components | shadcn/ui (new-york style) |
| Backend client | PocketBase JS SDK |
| Testing | Vitest + React Testing Library |
| Linting | ESLint (flat config) + Prettier |

## Usage

Once installed, trigger the skill in Claude Code:

> "Scaffold a new project" / "Start me a new app" / "Initialise this repo"

The skill autonomously runs all shell commands, writes config files, installs dependencies, and leaves you with a working dev server.

## Requirements

- Node.js ≥ 20
- An empty target directory
