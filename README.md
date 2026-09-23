# Maeve Baksa — Portfolio

Hugo portfolio site for Maeve Baksa.

## Design

- **Main theme:** [Contour](https://github.com/mobiusone-org/hugo-contour), pinned as a Git submodule.
- **Homepage:** deliberately close to Contour's default landing page, with one portrait treatment added.
- **Content pages:** mostly Contour, with restrained editorial details inspired by Digio.
- **Photography:** a dedicated, simplified full-screen gallery derived from [Bridget](https://github.com/Sped0n/bridget). Bridget is pinned as a second submodule for provenance/reference; its heavier SolidJS application is not loaded on the engineering site.
- **Favicon:** Lucide's open-source `cog` icon. See `THIRD_PARTY_LICENSES.md`.

## Local development

```bash
git clone --recurse-submodules https://github.com/maevebaksa/website.git
cd website
hugo server -D
```

This site is tested with Hugo Extended 0.166.0 or newer.

## Photography

Add future photography files under `static/photography/`, then add entries to `data/photography.toml`:

```toml
[[photo]]
src = "/photography/example.jpg"
alt = "Description of the photograph"
caption = "Optional caption"
```

## Contact

Public email: `hi@maevebaksa.com`.

## Images

Project/profile imagery should live in `static/images/` and be referenced with site-local paths. Do not introduce new dependencies on the retired WordPress `wp-content` tree.
