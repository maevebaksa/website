# Maeve Baksa — Portfolio

Hugo portfolio site for Maeve Baksa.

## Design

- **Base theme:** [Contour](https://github.com/mobiusone-org/hugo-contour), pinned as a Git submodule.
- **Visual direction:** Digio-forward — monochrome, editorial, terminal-like metadata, compact bordered modules — while retaining Contour's terrain canvas, navigation system, article layout, light/dark switching, and search-friendly Hugo architecture.
- Other themes were treated only as secondary inspiration.

## Local development

```bash
git clone --recurse-submodules https://github.com/maevebaksa/website.git
cd website
hugo server -D
```

This site is tested with Hugo Extended 0.166.0 or newer.

## GitHub Pages

The repository includes `.github/workflows/pages.yml`. In GitHub:

1. Open **Settings → Pages**.
2. Under **Build and deployment**, select **GitHub Actions** as the source.
3. Push to `main` or run **Deploy Hugo to GitHub Pages** manually.

The development URL is configured as:

`https://maevebaksa.github.io/website/`

## Moving to maevebaksa.com later

When ready to switch the domain:

1. Change `baseURL` in `hugo.toml` to `https://www.maevebaksa.com/`.
2. Add the custom domain in **Settings → Pages**.
3. Add the required DNS records at the DNS provider.
4. Add a `static/CNAME` file containing the final hostname after the DNS target is decided.

## Content migration

The site structure, CV, project list, and technical notes were migrated from the WordPress site. WordPress remains the source of the current machine images during the first GitHub Pages stage; those files can be copied into this repository before the WordPress host is retired.

## Pull requests

Trusted PRs from `maevebaksa` and Dependabot are configured for **auto-merge after checks**. GitHub does not count a PR author's self-approval as a valid independent review, so this avoids pretending self-approval can satisfy a required-review rule.
