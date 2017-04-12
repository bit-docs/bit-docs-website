@page plugins Plugins
@parent BitDocs 0
@description The bit-docs plugin system explained, and core plugins.

@body

The `bit-docs` plugin system provides hooks that allow you to create custom-generated content specific to your site, without the need to modify the `bit-docs` source itself.

In this section, you will find plugins maintained by the bit-docs organization; some of these plugins provide core functionality, though all can be replaced if they do not suit your needs. Please [let us know]() if you write your own plugins!

Continue reading, and we'll explore the architecture of a typical `bit-docs` plugin.

## Essentials of a Plugin

Every `bit-docs` plugin contains at least these two files:

- `bit-docs.js`
- `package.json`

Where `bit-docs.js` and `package.json` are for registering hooks and specifying dependencies, or other configuration, respectively.

Depending on the the plugin, you might also have files like:

- `tags.js`
- `theplugin.js`

Where `tags.js` or `theplugin.js` contain various functionalities of the plugin.

## Registering a Plugin

To use a plugin within a `bit-docs` enabled project, you must tell the project about the plugin.

In the case of a plugin named `the-plugin`, you would add it to a project's `package.json` like so:

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

When `bit-docs` attempts to load any plugin, it looks to require `bit-docs.js` from the plugin root. The `bit-docs.js` file is what bootstraps the plugin within the `bit-docs` system.

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

Let's break it down, starting with that first call to `bitDocs.register`.

#### Registering front-end dependencies

You will see this pattern in most `bit-docs` plugins that need to register front-end dependencies: 

```js
    var pkg = require("./package.json");
    var dependencies = {};
    dependencies[pkg.name] = pkg.version;
```

This simply gets the plugin name and version from `package.json`, and sets that as a key/value pair on an empty object, resulting in that familiar npm dependency format:

```json
{
    "the-plugin": "0.0.1"
}
```

Indeed, this object will be passed as an argument to a hook function and used to install this plugin on the front-end (more on this later).

Next, the `register` function of `bitDocs` is called (where `bitDocs` is an argument passed in by the `bit-docs` system at runtime):

```js
    bitDocs.register("html", {
        dependencies: dependencies
    });
```

In this example, `the-plugin` is registering itself with the `html` hook.

As stated earlier, the `html` hook is not handled by `bit-docs` itself ([look here](https://github.com/bit-docs/bit-docs/blob/master/lib/configure/configure.js#L43-L62) for the hooks `bit-docs` does handle internally). When `bit-docs` encounters a hook that it doesn't handle, it takes the arguments and passes them on to any plugin that might wish to handle that hook, such as [`bit-docs-generate-html`](https://github.com/bit-docs/bit-docs-generate-html).

The way `bit-docs-generate-html` handles the `html` hook is by calling `bitDocs.handle` in its own `bit-docs.js` file:

```js
module.exports = function(bitDocs){
    // ...
    bitDocs.handle("html", function(siteConfig, htmlConfig) {
        // ...
	});
};
```

Handling custom hooks like this will be covered later, but for now just know that's how the generator plugin `bit-docs-generate-html` informs `bit-docs` that it wants to handle any `html` hook, such as the one specified in `the-plugin`.

The `bit-docs-generate-html` plugin will download all those `dependencies` on the options objects hooked into `html` by plugins needing to load some dependencies on the front-end, such as `the-plugin`. It will then extract the specified front-end code into the generated website. The JavaScript or styles that `bit-docs-generate-html` loads into the front-end is determined by looking at the `main` value of the `package.json` of the dependency plugins that got downloaded (such as `the-plugin`'s `package.json`).

In the case of the example plugin, `the-plugin`, its `package.json` might look like:

```json
{
  "name": "the-plugin",
  "version": "0.0.1",
  "description": "An example plugin",
  "main": "theplugin.js",
  ...
```

In this case, the `main` file is set to `theplugin.js`. This `main` file should contain a `module.export` with the code that is intended to run on the front-end of the generated website.

For instance, `theplugin.js` might look like:

```js
module.exports = function(){
    alert(document.title);
};
```

Here are some low-level details about how `bit-docs-generate-html` does the extraction:

To "extract" that code from the `main` file of the plugin, and get it into the front-end of the generated website, `bit-docs-generate-html` composites all such front-end code from each registered "dependencies" plugin package, generating something like the following:

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

Notice the alert code has been extracted into this file.

You do not necessarily need to concern yourself with the details of such generated code (that's `bit-docs` job after all, to abstract such implementation details away from you); just know that `bit-docs-generate-html` places such code within the generated site, in that file: `static/bundles/bit-docs-site/static.js`, and that's how a `main` specified in a plugin `package.json` gets loaded to the front-end website!

We should point out that it's our other tool, [StealJS](http://stealjs.com), enabling the use of "modules" and `define` syntax on the front-end in this case.

This generated file is subsequently loaded on the front-end of the generated website using a normal script tag, and so in the case of the `the-plugin` example (with its `theplugin.js` code extracted), the alert message would appear on initial page load!

So, to recap on loading front-end depencies, all you need to do is include the `bit-docs-generate-html` plugin for your website project (almost every `bit-docs` enabled website will want to include this plugin, unless you create your own primary generator plugin), and then simply register the values of your plugin's `package.json` as shown for the `html` hook, making sure to point `main` to a file contianing the code you want to run on the front end!

##### Adding Less

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

[StealJS](http://stealjs.com) compiles and loads those styles to the front-end of the generated website.

#### Registering back-end dependencies

TBD

### Loading a plugin

TBD

### Writing a plugin

TBD
