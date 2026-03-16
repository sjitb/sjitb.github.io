# Content Organization

This directory contains site content grouped by section.

## Sections

- `about/` for biography and profile pages
- `projects/` for portfolio and case-study pages
- `blog/` for writing and updates
- `resources/` for references and learning material

## Adding A New Page

1. Create the page in the appropriate section directory.
2. Add YAML front matter with `layout`, `title`, `permalink`, and `nav_order`.
3. If the page belongs under a section, set `parent` to the section title.
4. Use lower-case hyphenated filenames.

## Navigation

- Section landing pages should be `index.md` and usually include `has_children: true`.
- Child pages should include `parent` and a spaced `nav_order` value such as `10`, `20`, `30`.
- Keep explicit `permalink` values to avoid URL changes during future reorganizations.
