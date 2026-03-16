# Satrajit Basu - Personal Website

This repository contains the source for my personal site built with Jekyll and the Just-the-Docs theme.

## Site Structure

- `index.md`: Home page
- `docs/about/`: About landing and profile pages
- `docs/projects/`: Projects landing and project pages
- `docs/blog/`: Blog landing and posts
- `docs/resources/`: Resources landing and reference pages

For content organization details, see `docs/README.md`.

## Local Development

1. Install dependencies:

   ```bash
   bundle install
   ```

2. Start a local preview server:

   ```bash
   bundle exec jekyll serve
   ```

3. Open `http://localhost:4000`.

## Adding New Content

1. Choose the appropriate section under `docs/`.
2. Create a Markdown file with YAML front matter.
3. Set `layout`, `title`, `permalink`, and `nav_order`.
4. For child pages, set `parent` to the section title.

## Deployment

This site is configured for GitHub Pages using the repository workflow in `.github/workflows/`.

## License

This repository is licensed under the MIT License. See `LICENSE`.
