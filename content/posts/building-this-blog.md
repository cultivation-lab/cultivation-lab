---
title: "How I Built This Blog for Free with Hugo and GitHub Pages"
date: 2026-08-08
draft: false
tags: ["meta", "hugo", "github-pages", "how-to"]
---

If you're reading this, it's coming to you from a blog that costs me exactly nothing to run. No hosting bill, no subscription, no "your free trial is ending" emails. Just some Markdown files in a folder, a static site generator called Hugo, and GitHub quietly serving it all to the internet for free.

I wanted to write down exactly how I got here — commands and all — partly so I remember it, and partly in case you're staring down the same "I guess I need a website now" feeling I started with. Fair warning: I hit four snags along the way. I've left them in, because knowing they're coming is most of the battle.

## The idea

You can host a real blog for free — not a locked-down placeholder, an actual site you own. The trick is a **static site generator plus GitHub Pages**: I write posts as plain Markdown, Hugo turns them into a fast static website, and GitHub Pages hosts the result for nothing. Everything lives in a Git repository, so my writing is fully portable.

I picked **Hugo** because it's a single program with no dependencies to babysit, the builds are ridiculously fast, and it was built for exactly this.

## Installing Hugo (and snag #1)

My first attempt at installing Hugo gave me a version that was years out of date, and my theme flatly refused to work with it. So I removed it and installed the current *extended* build straight from Hugo's releases instead:

```bash
sudo apt remove hugo
wget -O hugo.deb https://github.com/gohugoio/hugo/releases/download/v0.155.3/hugo_extended_0.155.3_linux-amd64.deb
sudo dpkg -i hugo.deb
rm hugo.deb
```

Then I checked it — you want a recent version number *and* the word `extended`:

```bash
hugo version
```

(That URL is for standard 64-bit Intel/AMD. On an ARM machine, swap `linux-amd64` for `linux-arm64`.)

## Creating the site

```bash
hugo new site cultivationblog --format yaml
cd cultivationblog
git init
```

The `--format yaml` gives you a `hugo.yaml` config, which is friendlier to read than the default.

## Adding a theme

I used **PaperMod**, added as a Git submodule — that's what lets the automated build fetch the theme later:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Then I set up `hugo.yaml`. This is also where **snag #2** lives: on GitHub Pages, your repo name controls your URL. I named my repo normally rather than the one magic name that gives a clean address, so my site lives at a path with the repo name in it — which means that path has to go into `baseURL`, or the styling breaks:

```yaml
baseURL: "https://cultivation-lab.github.io/cultivation-lab/"
languageCode: "en-us"
title: "My Blog"
theme: "PaperMod"
```

## Writing the first post

```bash
hugo new content posts/hello-world.md
```

Then I previewed everything locally — the `-D` flag includes drafts:

```bash
hugo server -D
```

Opening `http://localhost:1313` showed the site running with the theme applied.

## Pushing to GitHub (and snag #3)

I created an empty repository on github.com, then connected my local folder to it and pushed:

```bash
git add .
git commit -m "Initial Hugo blog"
git branch -M main
git remote set-url origin https://github.com/cultivation-lab/cultivation-lab.git
git push -u origin main
```

Then I added the automation file that rebuilds and redeploys the site on every push:

```bash
mkdir -p .github/workflows
```

Inside `.github/workflows/gh-pages.yml`:

```yaml
name: GitHub Pages
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-22.04
    permissions:
      contents: write
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.155.3'
          extended: true
      - name: Build
        run: hugo --minify
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v4
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

And here's snag #3: the moment my push included that workflow file, GitHub rejected the whole thing, complaining that my credentials lacked a `workflow` scope. Pushing anything into the automation folder needs that extra permission. One command fixed it:

```bash
gh auth refresh -h github.com -s workflow
git push
```

## Turning on Pages

Over on GitHub, I opened the **Actions** tab and waited for the build to go green. That first successful run creates a `gh-pages` branch holding the built site. Then in **Settings → Pages**, I set the source to **Deploy from a branch**, picked **gh-pages** and **/ (root)**, and saved.

## The post that vanished (snag #4)

The site deployed, the theme looked perfect, and my post was nowhere. Classic Hugo: posts marked as *drafts* show up in local preview but get skipped by the real build. I opened the post, flipped one line in the frontmatter from `draft: true` to `draft: false`, added some actual body text below it, and pushed:

```bash
git add .
git commit -m "Publish hello world post"
git push
```

It appeared.

## The whole loop, from here on

That was the setup. Publishing anything new is now just:

```bash
hugo new content posts/my-next-post.md
# write it, set draft: false
git add .
git commit -m "New post"
git push
```

About thirty seconds after the push, it's live. Total cost: nothing. The only thing I could ever choose to pay for is a custom domain someday, which would slot right in without moving a single file.

If you've been meaning to start a blog and "where do I even put it" has been stopping you — this is a very good answer, and those four snags above are basically the entire difficulty curve. Now you know all of them in advance.

More soon.
