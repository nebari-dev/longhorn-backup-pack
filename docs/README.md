# Nebari Longhorn Backup Pack Documentation

This directory contains the [Docusaurus](https://docusaurus.io/) site for the Nebari Longhorn Backup Pack. The site is written in TypeScript.

## Prerequisites

- Node.js `>= 20` (enforced by the `engines` field in `package.json`).
- npm (bundled with Node.js).

## Install

```bash
cd docs
npm install
```

## Local development

```bash
npm start
```

Starts the Docusaurus dev server with hot reload on http://localhost:3000/.

Note: the lunr search index is generated only by `npm run build`. The search box in the dev server will return no results; use a production build to exercise search.

## Production build

```bash
npm run build
```

Emits static files to `docs/build/`. The build step also produces the lunr search index via `docusaurus-lunr-search`.

## Preview the production build

```bash
npm run serve
```

Serves the contents of `docs/build/` locally so you can verify the production output, including search.

## Type checking

```bash
npm run typecheck
```

## CI

The [`Deploy docs to GitHub Pages` workflow](../.github/workflows/deploy-docs.yml) builds the site for every pull request and push to `main`, and deploys to GitHub Pages on merge to `main`.

To enable GitHub Pages for this repository, go to **Settings → Pages** and set the source to **GitHub Actions**.
