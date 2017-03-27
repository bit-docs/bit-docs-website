bit-docs
    receives
        finder
        generate
        processor

    options
        dependencies


bit-docs-generate-html
    options
        html
        "minifyBuild": false,
        "dest": "./gh-pages"
        parent

    provides
        generate
        tags
            templateRender

    receives
        html

    modules
        bit-docs-generate-html/build/renderer
        bit-docs-generate-html/build/static_dist

bit-docs-finder
    options
        glob

    provides
        finder
