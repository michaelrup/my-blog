# My Blog

A personal blog built with [Hugo](https://gohugo.io/) using the [hugo-blog-awesome](https://github.com/hugo-sid/hugo-blog-awesome) theme.

## Local Development

```bash
# Start development server with drafts enabled
hugo server -D

# Build for production
hugo
```

## Deploying to GitHub Pages with a Custom Domain

### 1. Create the GitHub Actions Workflow

Create `.github/workflows/hugo.yaml`:

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.152.2
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: Europe/Berlin
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 2. Configure GitHub Repository Settings

1. Push your code to GitHub
2. Go to **Settings → Pages**
3. Under "Build and deployment", set **Source** to **GitHub Actions**

### 3. Set Up Your Custom Domain

#### In GitHub:
1. Go to **Settings → Pages**
2. Under "Custom domain", enter your domain (e.g., `blog.example.com`)
3. Enable **Enforce HTTPS**

#### In Your DNS Provider:
For an apex domain (e.g., `example.com`), add these A records:
```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

For a subdomain (e.g., `michael.rupprecht.io`), add a CNAME record:
```
michael.rupprecht.io -> yourusername.github.io
```

#### In Your Repository:
A `static/CNAME` file has already been created with the domain:
```
michael.rupprecht.io
```

This file will be copied to the root of your published site during build.

### 4. Verify hugo.toml

The `baseURL` in `hugo.toml` is already configured:

```toml
baseURL = 'https://michael.rupprecht.io/'
```

### 5. Deploy

Commit and push your changes. The GitHub Action will automatically build and deploy your site.

```bash
git add .
git commit -m "Add GitHub Pages deployment workflow"
git push
```

Monitor the deployment in the **Actions** tab of your repository.

## Resources

- [Hugo Documentation](https://gohugo.io/documentation/)
- [Host on GitHub Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
- [Managing a custom domain for GitHub Pages](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site)
