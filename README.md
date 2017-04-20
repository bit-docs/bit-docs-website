# bit-docs-website

Website that documents bit-docs.

## Getting Started

### Install Dependencies

```shell
npm install
```

### Generate Website

```shell
npm run bit-docs
```

### Preview Generated Website

```shell
http-server gh-pages
```

Now visit <http://127.0.0.1:8080/>

### Publish Generated Website to GitHub

```shell
npm run gh-pages
```

Now visit <https://bit-docs.github.io/bit-docs-website>

## All Available Commands

### `npm run-script`

```shell
available via `npm run-script`:
  cache-bust # Shortcut to clearing out bit-docs-generate-html cache without using (slow) forceBuild.
    rm -rf ./node_modules/bit-docs/lib/configure/node_modules/bit-docs-generate-html/site/templates ./node_modules/bit-docs/lib/configure/node_modules/bit-docs-generate-html/site/static ./gh-pages
  bit-docs   # Generates the website.
    bit-docs
  gen        # Generates the website with debug output.
    bit-docs -d
  genf       # Generates the website with debug output and forceBuild.
    bit-docs -df
  gh-pages   # Publishes the website to GitHub Pages.
    gh-pages -d gh-pages
  gsl        # Prints a log graph highlighting submodule changes.
    git log --graph --oneline -U0 --submodule | grep -E '^[*| /\]+([0-9a-f]{7} |Submodule |> |$)'
  gsu        # Fetch latest remote changes for submodules and merge them in.
    git submodule update --remote --merge
  pub        # Shortcut to publish the website to GitHub Pages.
    npm run gh-pages
  see        # Preview the generated website using the globally installed http-server.
    http-server gh-pages
```
