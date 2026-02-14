# xxntti3n.github.io

Jekyll site for [xxntti3n.github.io](https://xxntti3n.github.io).

## GitHub Pages: fix "File not found"

If the live site shows **"File not found"** or **"does not contain the requested file"**:

1. In the repo go to **Settings → Pages**.
2. Under **Build and deployment**, set **Source** to **GitHub Actions** (not "Deploy from a branch").
3. Save. The workflow **Deploy Jekyll site to Pages** will run on push to `main` and deploy the built site.

Local build: `bundle install && bundle exec jekyll serve` → http://localhost:4000
