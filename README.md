# Zork Network™ Documentation Site

> Explore Your Money.

Developer documentation for the **Zork Network™** (system) and **Zorkcoin** (currency, **ZORK**).

## Local development

The site is a Jekyll project (GitHub Pages). Build and serve locally:

```bash
bundle install
bundle exec jekyll serve
```

### Canonical URL and social previews

Open Graph and Twitter Card meta tags use **absolute URLs** (`og:image`, `og:url`, etc.). Those are built from `url` and `baseurl` in `_config.yml`, which are set to the production host:

- **`url`:** `https://docs.zork.network`
- **`baseurl`:** `""`

If you need a different base while running `jekyll serve` (for example `http://127.0.0.1:4000`), add a small override file and merge it when serving:

**`_config_local.yml`** (example — do not commit secrets):

```yaml
url: "http://127.0.0.1:4000"
baseurl: ""
```

```bash
bundle exec jekyll serve --config _config.yml,_config_local.yml
```

Social crawlers will still use production URLs once the site is deployed; the override is mainly for local checks of paths and builds.

### Per-page social (OG / Twitter) overrides

Defaults live in `_config.yml` under `social` and `_includes/social-meta.html`. You can override per Markdown page in the front matter:

```yaml
---
title: My Page
description: "Short summary used for link previews and meta description."
image: /assets/img/my-page-card.jpg
---
```

- **`description`** — Plain text or Markdown; it is stripped to plain text for `twitter:description` and `og:description`.
- **`image`** — Site-relative path (starting with `/`). If omitted, `social.default_image` is used (see `_config.yml`). JPEG and PNG are both fine.

Optional site-wide handles (uncomment or set in `_config.yml`):

```yaml
social:
  twitter_site: "@YourHandle"    # twitter:site
  twitter_creator: "@Author"   # twitter:creator
```

## Social preview assets

| File | Role |
|------|------|
| **`assets/img/og-default.jpg`** | Default share image (**1200×630**, JPEG). Referenced by `social.default_image` — small file for fast crawler fetches. |
| **`assets/img/og-default-1200.png`** | Same dimensions, **lossless PNG** (optional reference or non-JPEG contexts). |
| **`assets/img/og-default-original.png`** | Full-resolution source when editing artwork. |

### Regenerate the default JPEG from the original

Example using [ffmpeg](https://ffmpeg.org/) (quality scale here: **`-q:v 20`** — higher numbers ≈ smaller files; re-tune if banding appears):

```bash
ffmpeg -y -i assets/img/og-default-original.png \
  -vf "scale=1200:630:flags=lanczos" -q:v 20 assets/img/og-default.jpg
```

### Regenerate the lossless PNG (optional)

```bash
gem install chunky_png --user-install
export GEM_HOME="$(ruby -e 'puts Gem.user_dir')"
ruby -rchunky_png -rzlib -e "
  img = ChunkyPNG::Image.from_file('assets/img/og-default-original.png')
  img.resample_bilinear(1200, 630).save('assets/img/og-default-1200.png', compression: Zlib::BEST_COMPRESSION)
"
oxipng -o max --strip safe -a assets/img/og-default-1200.png
```

Use an oxipng binary built for your OS (the **aarch64-unknown-linux-musl** release often runs on older glibc hosts).

## Development Notes

At the moment there are some notes on the kHeavyHash that are in development
to better describe the algorithm. In addition Stratum notes with regards to the
kHeavyHash miners will be added.

---

*Hosted on GitHub Pages*

Many thanks to:
  - [jekyll-remote-theme](https://github.com/benbalter/jekyll-remote-theme)
  - [jekyll-theme-rtd](https://github.com/carlosperate/jekyll-theme-rtd)
