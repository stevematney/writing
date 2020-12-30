### CSS Modules with webpack

For our case, we didn't have an explicit need to build a Web Component with
React, webpack, and CSS Modules, but, given that it is such a common scenario,
we wanted to cover this case as well.

There are two key questions to be answered here to ensure that we conform to the
style constraints present with Web Components:

1. How can I ensure my styles end up as inline `<style>` tags?
2. How can I make sure the inline `<style>` tags are within the Shadow DOM of my
   Web Component?

We'll take these on as two separate pieces, but the key to both is utilizing
[`HtmlWebpackPlugin`](https://github.com/jantimon/html-webpack-plugin) to
generate an HTML file we'll use in our Web Component template. That plugin
should have a configuration similar to the following `webpack.config.js`. Note
`inject: false`; we need that because we won't be injecting our Web Component
javascript into the HTML template:

```javascript
const HtmlWebpackPlugin = require("html-webpack-plugin");
...
    new HtmlWebpackPlugin({
      template: "path/to/template.html",
      filename: "template.html",
      inject: false,
    }),
...
```

With that in place, we can work on inlining our styles.

#### Inlining Styles

The simplest way that we've found to get our styles inline in our Web Components
is a combination of two other webpack plugins; the
[`MiniCssExtractPlugin`](https://github.com/webpack-contrib/mini-css-extract-plugin)
and the
[`HTMLInlineCssWebpackPlugin`](https://github.com/Runjuu/html-inline-css-webpack-plugin).
The first step in that process is to add the loader from the
`MiniCssExtractPlugin` to your CSS loaders, and to include the plugin as part of
your plugins:

```javascript
const HtmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
...
  module: {
    rules: [
      {
        test: /\.css$/i,
        use: [MiniCssExtractPlugin.loader, 'css-loader'],
      },
    ],
    plugins: [
      new HtmlWebpackPlugin({
        template: "path/to/template.html",
        filename: "template.html",
        inject: false,
      }),
      new MiniCSSExtractPlugin()
    ]
  },
...
```

For the `HTMLInlineCSSWebpackPlugin`, it's helpful to look at the HTML for our
template:

```html
<!-- inline css -->
<div id="app-container">
  <div id="mount"></div>
</div>
```

In particular, `<!-- inline css -->` is key. This gives our
`HTMLInlineCSSWebpackPlugin` a target for where to place the inline styles. With
that, our `webpack.config.js` will look like this:

```javascript
const HtmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const HTMLInlineCSSWebpackPlugin = require("html-inline-css-webpack-plugin").default;
...
  module: {
    rules: [
      {
        test: /\.css$/i,
        use: [MiniCssExtractPlugin.loader, 'css-loader'],
      },
    ],
    plugins: [
      new HtmlWebpackPlugin({
        template: "path/to/template.html",
        filename: "template.html",
        inject: false,
      }),
      new MiniCSSExtractPlugin(),
      new HTMLInlineCSSWebpackPlugin({
        replace: {
          target: "<!-- inline css -->",
          removeTarget: true,
        },
      })
    ]
  },
...
```

The addition of the `HTMLInlineCSSWebpackPlugin` causes the final state of our
loaded CSS to replace `<!-- inline css -->` in our original template.

Altogether, those plugins solve the problem of placing the final result of our
CSS `import`s into our HTML template as `<style>` tags. That done, how do we use
the final HTML result in our Web Component?

#### Using The HTML Template In Our Web Component

The first idea for using this HTML might be to include it in your global HTML
page as a`<template>`, but we wanted to avoid this, as we didn't want dependents
of our Web Components to need more than a single resource for the Component to
work.

That being the case, our first pass of solving this problem simply pulled our
final HTML template into our JavaScript as a separate request once our Web
Component was initialized, but we thought — since we're using webpack, and
webpack knows about our final HTML — we should be able to inject the final
template as a string into our JavaScript.

After a lot of searching, we couldn't find any existing way to do this with
webpack, so we wrote a plugin to handle it. The
[`InlineHTMLTemplatePlugin`](https://github.com/WTW-IM/inline-html-template-plugin).
This plugin allows us to put a string into our JavaScript that webpack will
overwrite with our final HTML template. Including that in our
`webpack.config.js` would look like this:

```javascript
const HtmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const HTMLInlineCSSWebpackPlugin = require("html-inline-css-webpack-plugin").default;
const InlineHTMLTemplatePlugin = require("inline-html-template-plugin").default;
...
  module: {
    rules: [
      {
        test: /\.css$/i,
        use: [MiniCssExtractPlugin.loader, 'css-loader'],
      },
    ],
    plugins: [
      new HtmlWebpackPlugin({
        template: "path/to/template.html",
        filename: "template.html",
        inject: false,
      }),
      new MiniCSSExtractPlugin(),
      new HTMLInlineCSSWebpackPlugin({
        replace: {
          target: "<!-- inline css -->",
          removeTarget: true,
        },
      }),
      new InlineHTMLTemplatePlugin()
    ]
  },
...
```

And in our JavaScript, we simply place:

```javascript
const loadedTemplate = "/* InlineHTML: template.html */";
```

Where `template.html` is the `filename` of our HTML from the
`HtmlWebpackPlugin`.

Our final Incrementer app from above using this template inside
`ReactHTMLElement`'s constructor would look like this:

```javascript
import React from "react";
import ReactDOM from "react-dom";
import ReactHTMLElement from "react-html-element";

import WebComponentApp from "./WebComponentApp";

const loadedTemplate = "/* InlineHTML: template.html */";

class WebComponent extends ReactHTMLElement {
  connectedCallback() {
    ReactDOM.render(
      <React.StrictMode>
        <WebComponentApp />
      </React.StrictMode>,
      this.mountPoint
    );
  }

  constructor() {
    super(loadedTemplate, "#mount");
  }
}

customElements.define("web-component", WebComponent);
```

In our resulting JS bundle, this `InlineHTML` string will be replaced with our
HTML template. This allows us to use our final HTML as the `innerHTML` for our
component's Shadow DOM, and to mount our app to the `div` whose ID is `mount`.

When we load the HTML into our JavaScript, we avoid a second request, and this
allows our Web Component to be a single JavaScript bundle instead of requiring
downstream dependents to have to load more than one thing in order to use our
Web Components.

This allows us to isolate our styles to our Web Component, but what about fonts?
Don't they have to be set globally?
