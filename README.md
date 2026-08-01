<h1 align=center>Periyar International</h1>

<p align="center">
  Official website for <a href="https://periyarinternational.org">Periyar International</a> — promoting the humanistic ideals of
  Periyar E.V. Ramasamy (self-respect, social justice, and rational thinking) through chapters, events, and programs worldwide.
</p>

---

## About this site

This repository holds the source for [periyarinternational.org](https://periyarinternational.org): the organization's
homepage, event pages, blog posts, and volunteer/support information.

- **Blog & event content** lives under `content/english/blog/`. Posts double as event pages
  (flagged with `is_event: true` in frontmatter) — recurring themes include youth programs and contests (e.g. Drawing
  Contest, Sing a Song, Speech Competition, Short Essay Contest, Youth AI Challenge, Kahoot Quiz), community events
  (Run for Periyar, Ask the Chief Guest), and support/volunteering pages (donate, attend events, start a chapter).
- **Author profiles** live under `content/english/author/`.
- **Site-wide data** (about, team, contact, portfolio, etc.) lives under `data/en/`.

## Publishing content

Blog posts and events can be added or edited from a browser, without touching Markdown/YAML directly, via a
lightweight CMS admin panel: **https://periyar-admin.netlify.app/admin/**

- Sign in with GitHub. Saving a post commits directly to this repo's `main` branch, which triggers the deploy below.
- Publish access is controlled by GitHub collaborator (write) access on this repo — not by anything in the CMS itself.
- The admin panel's source lives in a separate private repo (`theperiyar/periyar-admin`), kept out of this public
  repo intentionally.

## Local development

```bash
# one-time project setup (pulls theme assets into place)
npm run project-setup

# start local dev server
npm run dev
```

Requires [Hugo (extended)](https://gohugo.io/getting-started/installing/) `v0.115.1` and Go `1.20.5` — see
`.github/workflows/main.yml` for the exact versions used in CI.

## Deployment

Every push to `main` triggers `.github/workflows/main.yml`, which builds the site with Hugo and force-pushes the
output to the `gh-pages` branch. GitHub Pages serves that branch at the custom domain configured in `CNAME`.

## Tech stack & references

- Built with the [Hugo](https://gohugo.io) static site generator, using the
  [Meghna Hugo theme](https://github.com/themefisher/meghna-hugo) by Themefisher/Gethugothemes.
- [periyarinternational.org](https://periyarinternational.org) — live site
- [facebook.com/periyarusa](https://www.facebook.com/periyarusa/) · [YouTube](https://www.youtube.com/@PeriyarIntl)

<!-- licence -->
## License

**Code license:** Released under the [MIT](LICENSE) license (inherited from the Meghna Hugo theme).

**Image license:** Images are used for this organization's purposes and are not covered by the code's MIT license.
