@page plugins Plugins
@parent BitDocs 0
@description The bit-docs plugin system explained, and core plugins.

@body

The `bit-docs` plugin system provides hooks that allow you to create custom-generated content specific to your site without the need to modify the `bit-docs` source itself. In this section, you will find plugins maintained by the bit-docs organization; some of these plugins provide core functionality, though all can be replaced if they do not suit your needs. Please [let us know]() if you write your own plugins!

Here we'll explore the architecture of a typical `bit-docs` plugin.

## Essentials

At the very least, a basic `bit-docs` plugin will contain these files:

- `bit-docs.js`
- `package.json`
- `tags.js`
- `theplugin.js`

Where `bit-docs.js` and `package.json` are for setup, and `tags.js` or `theplugin.js` might contain actual functionality.

### Registration

To use a plugin within a `bit-docs` enabled project, you must tell the project about your plugin.

In the case of a plugin named `the-plugin`, you would add to a project's `package.json` like:

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
      "the-plugin": "0.0.1"
    },
    ...
  }
}
```

When `bit-docs` attempts to load any such plugin, it looks to require a `bit-docs.js` file from the plugin root, to get the configuration and registration details. This `bit-docs.js` file is what bootstraps the plugin within the `bit-docs` system.

A typical `bit-docs.js` will contain basic setup like:

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

The first call to `bitDocs.register` in the code of that example `bit-docs.js` file above is registering code to be loaded on the front-end; plugins typically need to add front-end JavaScript or CSS to the generated website for user interface purposes. The `html` hook actually passes through the `bit-docs` system, and is subsequently handled by the default generator plugin, `bit-docs-generate-html`.

The way `bit-docs-generate-html` handles this hook is to download your plugin from npm at build time, and then extract code from specific files into the generated website. More on the specifics of this "extraction" later, but first let's dissect the registration code itself.

You will see this pattern in most of the plugins that need to register front-end dependencies: get the plugin name from the plugin's own `package.json`, and then set that as a key on an empty object with the value set to the plugin's current version, resulting in an object with that familiar npm dependency format:

```json
{
    "the-plugin": "0.0.1"
}
```

Next, the `register` function is called on `bitDocs`, where `bitDocs` is an argument passed into the default `module.exports` function by the `bit-docs` system at runtime. In this example, `the-plugin` is registering itself with the `html` hook. You might also have noticed the `tags` hook but ignore it for now; that registers a back-end dependency, covered in the next section.

As stated earlier, the `html` hook is not handled by `bit-docs` itself ([look here](https://github.com/bit-docs/bit-docs/blob/master/lib/configure/configure.js#L43-L62) for the hooks `bit-docs` does handle internally). When `bit-docs` encounters a hook that it doesn't handle, it takes the arguments and passes them on to any plugin that might wish to handle that hook, such as `bit-docs-generate-html`.

The way `bit-docs-generate-html` handles the `html` hook is by calling `bitDocs.handle` in its own `bit-docs.js` file:

```js
module.exports = function(bitDocs){
    // ...
    bitDocs.handle("html", function(siteConfig, htmlConfig) {
        // ...
	});
};
```

Handling custom hooks like this will be covered later, but for now just know that's how the default generator plugin `bit-docs-generate-html` will get to handle the `html` hook specified in `the-plugin`.

Once `bit-docs-generate-html` has downloaded all `dependencies` hooked into `html` by all plugins such as the example `the-plugin`, it will extract the specified front-end code into the generated website. The specified front-end code to extract comes from the `main` value of the downloaded plugin `package.json`.

In the case of the example plugin `the-plugin`, its `package.json` might look like:

```json
{
  "name": "the-plugin",
  "version": "0.0.1",
  "description": "An example plugin",
  "main": "theplugin.js",
  ...
```

In this case, the `main` file is `theplugin.js`. The `main` file of a plugin added to the `html` hook for `bit-docs-generate-html` should contain a `module.export` with whatever code that is intended to run on the front-end of the generated website.

For instance, `theplugin.js` might look like:

```js
module.exports = function(){
    alert(document.getElementById("demo").innerHTML);
};
```

To "extract" that code from the `main` file of the plugin and get it into the front-end of the generated website, `bit-docs-generate-html` composites all such front-end code from each registered "dependencies" package and generates the following code:

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

You do not necessarily need to concern yourself with the details of such generated code (that's `bit-docs` job after all, to abstract the implementation details of the website away from you); just know that `bit-docs-generate-html` places such code within the generated site in that file `static/bundles/bit-docs-site/static.js`, and that's how `main` specified in a plugin `package.json` gets loaded to the front-end.

However, we will point out that our other very useful tool, [StealJS](http://stealjs.com), is what enables the use of modules and `define` syntax, and that is also what handles the client-side loading of modules for the generated website as in the code above.

This file is subsequently loaded on the front-end of the generated website using a normal script tag, and so in the case of this `the-plugin` example with its `theplugin.js`, the alert message would appear on the initial page load!

So, to recap with loading front-end depencies, you need to include the `bit-docs-generate-html` plugin to the website project (almost every `bit-docs` enabled website will want to include this plugin), and then simply register the values of your plugin's `package.json` as shown to the `html` hook, making sure to point `main` to a file contianing the code you want to run on the front end!

##### Adding Less

If your plugin needs to add some CSS styles to the front-end of the generated website, you can do so by requiring a Less file in the `main` file mentioned above. For example, doing so for `theplugin.js` would look like:

```js
require("./thestyles.less")

module.exports = function(){
    alert(document.getElementById("demo").innerHTML);
};
```

With a Less file `thestyles.less` having contents like:

```css
body {
    div {
        background: red;
    }
}
```

Thanks again to [StealJS](http://stealjs.com), those styles will be compiled and loaded to the front-end of the generated website.

#### Registering back-end dependencies

TBD

### Loading a plugin

TBD

### Writing a plugin

TBD
