---
description: "Use when creating or editing Jekyll site pages in this repository. Enforces Just-the-Docs front matter consistency, concise professional tone, and safe Markdown structure."
name: "Jekyll Content Conventions"
applyTo: "**/*.md"
---

# Jekyll Content Conventions

- Prefer valid YAML front matter at the top of each page, wrapped in `---` markers.
- For standard pages, prefer including `layout`, `title`, `permalink`, and `nav_order`.
- For the home page (`index.md`), prefer `layout: home` with `title` and `nav_order`.
- Prefer existing permalink style with trailing slash for content pages (example: `/about/`).
- For section landing pages, prefer `has_children: true` when child pages are expected.
- For child pages, prefer `parent: <Section Title>` and spaced `nav_order` values (`10`, `20`, `30`).
- Use `grand_parent` only when a third navigation level is needed.
- Prefer a clear, professional first-person voice suitable for a personal portfolio site.
- Keep content concise and scannable with short paragraphs and meaningful headings.
- Prefer Markdown over inline HTML unless Markdown cannot express the needed structure.
- Avoid removing existing front matter keys unless the user explicitly requests a structural change.
