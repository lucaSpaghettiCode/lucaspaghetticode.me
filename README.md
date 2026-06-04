# Luca — Personal Website

A minimalist, single-page portfolio + blog. Dark/light theme, dedicated pages
for every project and essay (hash-routed), and an icon-based tech stack.

## Structure

```
luca-site/
├── index.html        ← the whole site (markup, styles, scripts)
└── assets/
    └── luca.png       ← profile photo
```

Everything lives in `index.html`. There is no build step and no dependencies to
install — fonts load from Google Fonts over the network.

## Run locally

**Option A — just open it.** Double-click `index.html`. It works directly from
the file system.

**Option B — run a tiny local server** (recommended; avoids any browser
file:// quirks and matches how it'll behave when hosted):

```bash
cd luca-site

# Python 3
python3 -m http.server 8000

# …or Node
npx serve .
```

Then visit http://localhost:8000

## Deploy

It's a static site — upload the folder to any static host:
Netlify (drag-and-drop), GitHub Pages, Cloudflare Pages, Vercel, S3, etc.

## Editing

- **Content** (work, education, projects, writing, contact): edit the markup in
  the `<main class="home-view">` block and the per-page `<main class="detail-view">`
  blocks.
- **Theme colors**: the `:root[data-theme="dark"]` / `[data-theme="light"]`
  variables at the top of the `<style>`.
- **Logos**: each `.logo` badge currently shows a monogram. To use a real image,
  replace e.g. `<span class="logo">TUM</span>` with
  `<span class="logo"><img src="assets/tum.png" alt="TU München"></span>`.
- **Tech icons**: the `T` map inside the first `<script>` block — add an entry
  `Name: { i:'<path .../>' }` and reference it via `data-tech="Name"`.
