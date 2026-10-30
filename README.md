# Folcadi articles

Write-ups about building **Folcadi** — a language-exchange messenger.

- Published site: <https://frankmilvus.github.io/folcadi-articles/>
- Cross-posted to dev.to: <https://dev.to/frankmilvus>
- Archive of the pre-rename posts: <https://frankmilvus.github.io/tongues-articles/>

## Layout

- `articles/` — the posts (Markdown, one file per post: `YYYY-MM-DD-slug.md`)
- `images/` — cover images (generated with `devto/make_cover.py` in the main repo)

## Why a second repo

`tongues-articles` holds the posts published before the rename and stays online
unchanged: every dev.to post declares a canonical URL under that path, and
GitHub Pages does not redirect a renamed project site. New posts go here, so
their canonical URLs are stable under a name that matches the product.

## How this relates to the app repo

The app lives in a separate (private) repository. The publishing tooling is in
its `devto/` folder; this repo holds the public, indexable copy of the writing
and its images, which is also what the dev.to covers link to.
