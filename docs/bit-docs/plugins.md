@page plugins Plugins
@parent BitDocs 0
@description The bit-docs plugin system explained, and core plugins.

@body

The `bit-docs` plugin system provides hooks that allow you to modify your generated website.

A number of plugins are maintained by the bit-docs organization. Some of these plugins provide core functionality, and will need to be included with every project, unless they are replaced; typically you will need a "finder" plugin to pull in files, a "processor" plugin to process pulled in data, and then a "generator" plugin to generate the output.

So, a typical `bit-docs` enabled project will probably include:

- [bit-docs-glob-finder] — Finder
- [bit-docs-dev] — Commonly used tags
- [bit-docs-js] — Processor
- [bit-docs-generate-html] — Generator

Here we'll explore the architecture of a typical `bit-docs` plugin.

## Essentials of a Plugin

Every `bit-docs` plugin contains at least these two files:

- `bit-docs.js`
- `package.json`

Where `bit-docs.js` and `package.json` are for registering hooks and specifying dependencies.

Depending on the the plugin, you might also have files like:

- `tags.js`
- `theplugin.js`

Where `tags.js` or `theplugin.js` might contain various functionalities of the plugin.

## Using a Plugin

To use a plugin within a `bit-docs` enabled project, you must tell the project about the plugin.

In the case of a plugin named `the-plugin`, you would add it to a website project's `package.json` like so:

```json
{
  "name": "my-project",
  ...
  "devDependencies": {
    "bit-docs": "*"
  },
  "bit-docs": {
    "dependencies": {
      ...
      "bit-docs-generate-html": "*",
      ...
      "the-plugin": "0.0.1"
    },
    ...
  }
}
```

## Registering a Plugin

When `bit-docs` attempts to load any plugin, it looks to require `bit-docs.js` from the plugin root.

The `bit-docs.js` file is what bootstraps the plugin within the `bit-docs` system.

A typical `bit-docs.js` will register some hooks, for example:

```js
var tags = require("./tags");
module.exports = function(bitDocs){
    var pkg = require("./package.json");
    var dependencies = {};
    dependencies[pkg.name] = pkg.version;

    bitDocs.register("html", {
        dependencies: dependencies
    });

    bitDocs.register("tags", tags);
}
```

This might look intimidating if you're not familiar with the `bit-docs` plugin API, but it's straightforward.

### Registering front-end dependencies

You will see this pattern in most `bit-docs` plugins that need to register front-end dependencies: 

```js
    var pkg = require("./package.json");
    var dependencies = {};
    dependencies[pkg.name] = pkg.version;
```

This simply gets the plugin name and version from `package.json`, setting them as a key/value pair on an empty object, resulting in that familiar npm dependency format:

```json
{
    "the-plugin": "0.0.1"
}
```

Indeed, this object will be passed as an argument to a hook function, and used to install this plugin on the front-end.

Next, the `register` function of `bitDocs` is called (`bitDocs` is an argument passed in by `bit-docs` at runtime):

```js
    bitDocs.register("html", {
        dependencies: dependencies
    });
```

In this example, `the-plugin` is registering itself with the `html` hook.

As stated earlier, the `html` hook is not handled by `bit-docs` itself ([look here](https://github.com/bit-docs/bit-docs/blob/master/lib/configure/configure.js#L43-L62) for the hooks `bit-docs` does handle). When `bit-docs` encounters a hook that it doesn't handle, it takes the arguments and passes them along to any plugin that might wish to handle that hook, such as [`bit-docs-generate-html`](https://github.com/bit-docs/bit-docs-generate-html).

The way `bit-docs-generate-html` handles the `html` hook is by calling `bitDocs.handle` in its own `bit-docs.js` file:

```js
module.exports = function(bitDocs){
    // ...
    bitDocs.handle("html", function(siteConfig, htmlConfig) {
        // ...
    });
};
```

Handling custom hooks like this will be covered later, but just know that's how `bit-docs-generate-html` informs `bit-docs` that it wants to handle any `html` hook (such the one registered in `the-plugin`).

The `bit-docs-generate-html` plugin will use npm to download any `dependencies` set on the option objects passed as arguments to the `html` hook. The code that `bit-docs-generate-html` will load into the front-end is determined by looking at `package.json`'s `main` for the fetched dependency (such as `the-plugin/package.json`).

In the case of `the-plugin`, `main` in `package.json` would be set to `theplugin.js`:

```json
{
  "name": "the-plugin",
  "version": "0.0.1",
  "description": "An example plugin",
  "main": "theplugin.js",
  ...
```

`main` should have a `module.export` with code intended to run on the front-end of the generated website.

So, `theplugin.js` might look something like:

```js
module.exports = function(){
    alert(document.title);
};
```

To "extract" that code from the `main` file of the plugin, and get it into the front-end of the generated website, `bit-docs-generate-html` composites all front-end code from each registered dependency:

```js
/*the-plugin@0.0.1#theplugin*/
define('the-plugin@0.0.1#theplugin', function (require, exports, module) {
    module.exports = function(){
        alert(document.getElementById("demo").innerHTML);
    };
});
/*bit-docs-site@0.0.1#packages*/
define('bit-docs-site@0.0.1#packages', function (require, exports, module) {
    function callIfFunction(value) {
        if (typeof value === 'function') {
            value();
        }
        return value;
    }
    module.exports = {
        'the-plugin': callIfFunction(require('the-plugin'))
    };
});
```

The alert code has been extracted into this file, as would any other front-end code registered by any other plugins.

You do not necessarily need to concern yourself with the details of this generated code (that's `bit-docs` job after all, to abstract away such implementation details); just know that `bit-docs-generate-html` places this generated code within the generated site, in the file `static/bundles/bit-docs-site/static.js`, and that's how a `main` file specified in a plugin's `package.json` that registers itelf with the `html` hook gets loaded into the front-end website!

Note: It's our other tool, [StealJS](http://stealjs.com), enabling the use of "modules" and `define` syntax on the front-end.

The composite file is loaded on the front-end of the generated website using like a normal script tag by Steal. In this case, that means the alert message would appear on initial page load of the genereted website!

To recap on loading front-end depencies:

All you need to do is include the `bit-docs-generate-html` plugin for your website project (almost every `bit-docs` enabled website will want to include this plugin, unless you create your own primary generator plugin), and then simply register the `html` hook with the `name` and `version` values of your plugin's `package.json`, making sure `main` in `package.json` points to a file contianing a `module.exports` with the code you want to run on the front-end!

#### Adding Less

If your plugin needs to add some styling to the front-end of the generated website, you can do so by requiring a Less file in the `main` file mentioned above.

For example, doing this in `theplugin.js` might look like:

```js
require("./thestyles.less")

module.exports = function(){
    alert(document.getElementById("demo").innerHTML);
};
```

Then, create a Less file named `thestyles.less` having contents like:

```css
body {
    div {
        background: red;
    }
}
```

Now when you generate the website, it should have a red background!

Note: Again, [StealJS](http://stealjs.com) compiles and loads those styles to the front-end of the generated website.

### Registering back-end dependencies

TBD
