# blog publishing gateway

Public Zola site for Sasha and Drinkins's articles.

Canonical prose, research notes, bibliography sources, and publication adapters live in
the private `CopilotDrinkins/Blog` backend. Files under `content/posts/` are publication
projections and should not be edited directly.

## Local validation

The site and CI use Zola 0.22.1.

```bash
zola check
zola build
zola serve --interface 127.0.0.1 --port 8765
```

Add `--drafts` to `build` or `serve` when reviewing unpublished posts. Validate the
rendered product in Chromium and Firefox at desktop/laptop/mobile widths, including TOC
hierarchy, fragments, cuts, overflow, resources, and browser console/page errors.

## Deployment

A push to `main` runs `.github/workflows/deploy.yml`: GitHub Actions installs Zola
0.22.1, builds `public/`, and publishes it to `gh-pages`. A local build does not authorize
a commit or push.
