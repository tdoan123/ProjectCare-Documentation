# ProjectCare Documentation

Technical documentation site for **TeamYellow's ProjectCare** — an ASP.NET Core MVC Healthcare Client Management System. Built with [Docusaurus](https://docusaurus.io/) and deployed to GitHub Pages.

## Live Site

[https://tdoan123.github.io/ProjectCare-Documentation/](https://tdoan123.github.io/ProjectCare-Documentation/)

## Prerequisites

- [Node.js](https://nodejs.org/) v20 or higher
- npm (comes with Node.js)

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/tdoan123/ProjectCare-Documentation.git
cd ProjectCare-Documentation
```

### 2. Install dependencies

```bash
npm install
```

### 3. Start the local development server

```bash
npm start
```

This opens the site at `http://localhost:3000/ProjectCare-Documentation/` and hot-reloads on file changes.

## Adding or Editing Documentation

All documentation lives in the [`docs/`](docs/) folder as Markdown (`.md`) files. The sidebar is auto-generated from the folder structure.

| File | Content |
| ---- | ------- |
| `docs/intro.md` | Project overview |
| `docs/database-schema-fixed.md` | Database schema |
| `docs/controller-inventory.md` | Controller reference |
| `docs/subscription-flow-diagrams.md` | Subscription flow diagrams |
| `docs/usecase-system.md` | Use case system diagrams |

To add a new page, create a `.md` file in `docs/` — it will appear in the sidebar automatically.

## Build

```bash
npm run build
```

Generates a static site into the `build/` directory.

## Deployment

Deployment is automated via GitHub Actions on every push to `main`. The site is published to the `gh-pages` branch automatically — no manual steps needed.
