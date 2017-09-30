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
http-server -a localhost -p 8080 -c-1 gh-pages
```

_Note_: The `-a localhost -p 8080` are needed for the `-c-1` cache control flag to work.

Now visit <http://localhost:8080/>

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
    npm run clean-dest && rm -rf ./node_modules/bit-docs/lib/configure/node_modules/bit-docs-generate-html/site/templates ./node_modules/bit-docs/lib/configure/node_modules/bit-docs-generate-html/site/static
  clean-dest # Deletes the destination folder.
    rm -rf ${npm_package_bit_docs_dest:-./doc}
  bit-docs   # Generates the website.
    bit-docs
  gen        # Generates the website with debug output.
    bit-docs -d
  genf       # Generates the website with debug output and forceBuild.
    bit-docs -df
  gh-pages   # Publishes the website to GitHub Pages.
    gh-pages -d gh-pages
  install-modules # Installs the npm modules to be documented.
    npm install --prefix docs/modules
  pub        # Shortcut to publish the website to GitHub Pages.
    npm run gh-pages
  see        # Preview the generated website using the globally installed http-server.
    http-server -a localhost -p 8080 -c-1 gh-pages
```
