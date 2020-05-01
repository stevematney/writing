# The Big Guide to Web Components Built With React

Our team at WTW wanted to find a way to deploy multiple small applications to the same webpage in a way that allows greater than a dozen teams to separately develop, build, and deploy their apps in isolation, and without having to have multiple gates (other people, teams, etc) in order to get new code into production. (This approach is often called [Micro Frontends](https://micro-frontends.org/).)

We researched a lot of different way of handling this problem, but finally landed on utilizing Web Components as our solution. This allows us to deploy individual, isolated Javascript applications together on a single page without much responsibility on other teams to expend a lot of work to integrate those components.

The primary web technology our shop uses is React, so using React with our Web Components decision was a primary need for us. We've learned a lot as we've treaded the path of building isolated Web Components with React, and this article will detail much of that learning.

## What are Web Components?

[MDN has a very thorough and clear guide on Web Components](https://developer.mozilla.org/en-US/docs/Web/Web_Components), but in the simplest terms, Web Components is a way to build encapsulated elements rendered in isolation from the other elements on the DOM. This isolation allows you to have modularity and portability in individual client-side components, indifferent to their surrounding context.

This portability and isolation can enable many of the benefits of Micro Frontends, which is why we decided to pursue Web Components as a Micro Frontends solution. [We'll discuss this at length later.](#web-components-as-micro-frontends)

But if we're using React, isn't some of that portability and isolation already built in?

## Why Build Web Components With React? Why not just _use React_?

In a Web Component, you're not limited in the technology you use, so any approach should be feasible, but if you're using a technology like React already, why not simply ship shared React components for different pieces of functionality?

There are at a few answers to this question: 

1. Shared components within the React system depend on a common set of React functionality. If you're building a component, and you want to use a newer feature, but the global version of React in your system does not support this feature, then you simply can't use it. A Web Component has no need to have a top-down version of React dictating the features it can use. The Web Component can use the version it needs.

2. Evolvability: the isolation of Web Components allows to you use React or really any other technology you see fit. If your team decides that something better or more fitting has come along for your Web Component, you can change it without worrying about affecting the surrounding page. Or you can build a Web Component without any third party library at all, and it can integrate in any web page easily (even those built with React).

3. Possibly the most important answer is deployability. One of the main concerns that a Micro Frontends approach aims to alleviate is a lack of dependency on external teams when deploying an application. If you're using React to piece together your frontend, all of your components need to be up to date at build time. With a Web Component whose source is hosted on an independent server, you can deploy new source code (which we [go into detail about](#file-naming-cache-busting-and-deployment) below), and your downstream dependents will be updated automatically.

There are some limitations and hurdles when it comes to using React in Web Components, and some questions we wanted to answer as we approached the idea  Let's talk about those. The first is the problem of lost functionality.

## The Functionality Problem

The [React Docs describe a way you can use React to render Web Components](https://reactjs.org/docs/web-components.html#using-react-in-your-web-components), but a deeper exploration of this idea reveals an issue: anything that uses events within your React Web Component to update the internal state simply does not work. You can [see an example of this broken functionality on this Sandbox](https://codesandbox.io/s/react-web-component-without-retargeting-events-b61u3?file=/src/index.js), (displayed below):

<iframe
     src="https://codesandbox.io/embed/react-web-component-without-retargeting-events-b61u3?fontsize=14&hidenavigation=1&initialpath=%2Fhome&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="React Web Component Without Retargeting Events"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>


The solution to this problem is to retarget events to the Shadow DOM, rather than to the original root node of the mounted element (`document.body`). [This React issue](https://github.com/facebook/react/issues/9242) describes a few different ways to solve this problem, most notably [`react-shadow-dom-retarget-events`](https://github.com/spring-media/react-shadow-dom-retarget-events), which works by adding an event listener for react-specific events to the Shadow DOM. This method works, to an extent, but can potentially be brittle (like if React adds new events). We wanted a less potentially-brittle solution.

### ReactHTMLElement

Our approach has been a little bit different from `react-shadow-dom-retarget-events`: we wrote a Javascript Class called [`react-html-element`](https://github.com/WTW-IM/react-html-element) that extends `HTMLElement`, creates a Shadow DOM, adds the appropriate `createElement` functions to it,  and sets the Shadow DOM as the `ownerDocument` for the `mountPoint` where your React app is intended to be mounted. This solves the event retargeting problem in a robust way because it doesn't depend on any specific React implementation details, and simply allows the Shadow DOM to capture all events from within the React application. You can see the [same example from above, but using `ReactHTMLElement` ](https://codesandbox.io/s/react-web-component-with-reacthtmlelement-7gsrj?file=/src/index.js), and now the functionality is working. (Displayed below.)

<iframe
     src="https://codesandbox.io/embed/react-web-component-with-reacthtmlelement-7gsrj?fontsize=14&hidenavigation=1&initialpath=%2Fhome&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="React Web Component With ReactHTMLElement"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

This was our first consideration, as it is the most clear problem when building Web Components with React. The next problem we wanted to solve was: how do we render HTML Children within a Web Component?

## Rendering Externally-Provided Children

The Web Components APIs provide [a very simple way of rendering children using `<slot>` tags](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_templates_and_slots#Adding_flexibility_with_slots). Luckily, we found that using `<slot>` in React was easy, and it just works. You can manage `<slot>` tags with `name` attributes, but the absolute simplest way is just to include a bare `<slot>` tag in your React component:

```react
export default function AppWithSlot() {
  <div>
    Here are my children:
    <div>
      <slot></slot>
    </div>
  </div>
}
```

With that in place (and with your app registered as a Web Component called `react-app`), your HTML could look something like this:

```html
<html>
  <head>
    <!-- load your js, Web Components polyfills, etc -->
  </head>
  <body>
    <react-app>
      <div>
        Any children can go in here, including other Web Components, and even other Web Components built with React.
      </div>
    </react-app>
  </body>
</html>
```

That's it! A very simple solution built right into the browser!

The next question we wanted to answer was: how do Web Components communicate with each other?

## Inter-Component Communication

There are really two ways to handle this problem. You can  [build a Web Component with a JavaScript API](https://developers.google.com/web/fundamentals/web-components/customelements#jsapi), and pass data into the Web Component using a reference to the element itself, and calling the Javascript API you have defined. This method is good, but probably the simplest way for Web Components built with React is to use events on the Window. An app can emit a Window event, and another can listen for it to update based on any information that has changed.

Using events has a few key benefits:

* Eventing is a [robust way of transferring complex data objects](https://developer.mozilla.org/en-US/docs/Web/Guide/Events/Creating_and_triggering_events).
* The implementation of a listener or dispatcher can be close to the point where that event information is  utilized.
* Your components can dispatch an event without knowing what might be listening.
* Listeners can respond to events without knowing what element might dispatch it.

With that resolved, we could tackle our next challenge: styles.

## Styles

Styles in Web Components have some unique constraints:

* Styles specific to the component must be contained within the component itself
* `<link>` tags do not work within Web Components. Styles for the Component must be in `<style>` tags.
* `@font-face`s must be loaded on the global document, and cannot be loaded from within a Web Component.

There are other concerns, as well, when dealing with our particular build process. Most of our teams use a very common set of tools in the React community: webpack, Babel, and styled-components. These tools make bundling apps for the web very easy, but the above styling constraints bring an extra level of complexity. By default, styled-components applies your styles to the `<head>` of your HTML, but we need them applied inside the Shadow DOM.

### Styled Components

Luckily, styled-components gives us a simple solution for the problem of relocation style tags: the [`StyleSheetManager`](https://styled-components.com/docs/api#stylesheetmanager). This allows you to provide a node where the styles will ultimately land, so for a Web Component, you don't need to do much more than wrap your App in a stylesheet manager. A Web Component using [`ReactHTMLElement `](#reacthtmlelement) and `StyleSheetManager` might look like this:

```react
class ReactWebComponent extends ReactHTMLElement {
  connectedCallback() {
    ReactDOM.render((
      <StyleSheetManager target={this.mountPoint}>
        <App />
      </StyleSheetManager>
    ), this.mountPoint);
  }
}
```



### CSS Modules with webpack

For our case, we didn't have an explicit need to build a Web Component with React, webpack, and CSS Modules, but, given that it is such a common scenario, we wanted to cover this case as well.

There are two key questions to be answered here to ensure that we conform to the style constraints present with Web Components:

1. How can I ensure my styles end up as inline `<style>` tags?
2. How can I make sure the inline `<style>` tags are within the Shadow DOM of my Web Component?

We'll take these on as two separate pieces, but the key to both is utilizing [`HtmlWebpackPlugin`](https://github.com/jantimon/html-webpack-plugin) to generate an HTML file we'll use in our Web Component template. That plugin should have a configuration similar to the following `webpack.config.js`. Note `inject: false`; we need that because we won't be injecting our Web Component javascript into the HTML template:

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

The simplest way that we've found to get our styles inline in our Web Components is a combination of two other webpack plugins; the [`MiniCssExtractPlugin`](https://github.com/webpack-contrib/mini-css-extract-plugin) and the [`HTMLInlineCssWebpackPlugin`](https://github.com/Runjuu/html-inline-css-webpack-plugin). The first step in that process is to add the loader from the `MiniCssExtractPlugin` to your CSS loaders, and to include the plugin as part of your plugins:

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

For the `HTMLInlineCSSWebpackPlugin`, it's helpful to look at the HTML for our template:

```html
<!-- inline css -->
<div id="app-container">
  <div id="mount"></div>
</div> 
```

In particular, `<-- inline css -->` is key. This gives our `HTMLInlineCSSWebpackPlugin` a target for where to place the inline styles. With that, our `webpack.config.js`  will look like this:

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

The addition of the `HTMLInlineCSSWebpackPlugin` causes the final state of our loaded CSS replace the `<!-- inline css -->` comment is in our original template.

Altogether, those plugins solve the problem of placing the final result of our CSS `import`s inline in our HTML template. That done, how do we use the final HTML result in our Web Component?

#### Using The HTML Template In Our Web Component

The first idea for using this HTML might be to include it in your global HTML page as a`<template>`, but we wanted to avoid this, as we didn't want dependents of our Web Components to need more than a single resource for the Component to work.

That being the case, our first pass of solving this problem simply pulled our final HTML template into our JavaScript as a separate request once our Web Component was initialized, but we thought — since we're using webpack, and webpack knows about our final HTML — we should be able to inject the final template as a string into our JavaScript.

After a lot of searching, we couldn't find any existing way to do this with webpack, so we wrote a plugin to handle it. The [`InlineHTMLTemplatePlugin`](https://github.com/WTW-IM/inline-html-template-plugin). This plugin allows us to put a string into our JavaScript that webpack will overwrite with our final HTML template. Including that in our `webpack.config.js` would look like this:

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

Where `template.html` is the `filename` of our HTML from the `HtmlWebpackPlugin`. 

Our final Incrementer app from above using this template inside `ReactHTMLElement`'s constructor would look like this:

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
    super(loadedTemplate, "#mount")
  }
}

customElements.define("web-component", WebComponent);
```

In our resulting JS bundle, this `InlineHTML` string will be replaced with our HTML template. This allows us to use our final HTML as the `innerHTML` for our component's Shadow DOM, and to mount our app to the `div` whose ID is `mount`.

When we load the HTML into our JavaScript, we avoid a second request, and this allows our Web Component to be a single JavaScript bundle instead of requiring downstream dependents to have to load more than one thing in order to use our Web Components.

This allows us to isolate our styles to our Web Component, but what about fonts? Don't they have to be set globally?

### Fonts

Fonts _do_ need to be set globally. For our case, all of our pages start with a shop-wide page template, and are loaded with the appropriate fonts and an icon font anyway, so we don't have to do anything clever to expect that to work. If you don't have the luxury of expecting the fonts to be already on the page, and want to ensure that fonts used in your Web Component are available on pages that load them, we recommend injecting a `<link>` tag for your font to the `document.head` from within your Web Component JavaScript. A function to do that could look like this:

```javascript
function injectFont(sheet: string) {
  const foundSheet = document.querySelector(`link[href="${sheet}"]`);
  if (foundSheet) return;

  const link = document.createElement('link');
  link.setAttribute('rel', 'stylesheet');
  link.setAttribute('href', sheet);
  document.head.prepend(link);
}
```

Unfortunately, there doesn't seem to be a way to do this without interacting with the global `document`, but the function above will at least prevent loading the same stylesheet more than once.

With all those questions answered, we set out to implement some Web Components using our custom library of React Components, [`es-components`](https://github.com/WTW-IM/es-components), but in doing so, we discovered some other issues, particularly with things like Modals, which expect to be attached to the `document.body`.

## React Portals and document.body

Some of the components in `es-components` use `ReactDOM.createPortal` in order to inject dynamic content onto the page. Our implementation was initially unaware of Web Components, and simply used `document.body` as the target for those created elements. This had a problem when trying to utilize those components within a Web Component because the injected content would land outside our Shadow DOM, and would thus lose its styling. So how do we ensure our component does the right thing when inside a regular `document.body`, and also does the right thing when it lives within a Shadow DOM?

The key here, for us, was to use the [`getRootNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode) function on the `Node` prototype, but we couldn't just use it directly.

When `getRootNode` returns `document`, we needed to drill down to `document.body` because attempting  to append a child to the `document` directly produces the error "[Failed to execute 'appendChild' on 'Node': Only one element on document allowed](https://stackoverflow.com/a/48938528)."

In order to circumvent this error, we created a react hook called [`useRootNode`](https://github.com/WTW-IM/es-components/blob/master/packages/es-components/src/components/util/useRootNode.js) that appropriately returns `document.body`  when a component ref is on the `document` directly, or the correct shadow root when the ref is within a Shadow DOM. This allowed us to use the value from `useRootNode` to target our `React.createPortal` calls, rather than naively pointing only to `document.body`.

With that solved, we could talk about putting the picture together.

## Web Components as Micro Frontends

For our purposes, we wanted to deploy Web Components so that individual pieces of functionality could be deployed on the same page by different teams with only loose dependencies on each other. Micro Frontends are the architectural approach that allow us to solve that problem, and there are [many solutions](https://www.angulararchitects.io/post/2017/12/28/a-software-architect-s-approach-towards-using-angular-and-spas-in-general-for-microservices-aka-microfrontends.aspx), each with tradeoffs and benefits. For us, we had a few main goals:

* We wanted runtime dependencies rather than build-time dependencies. We didn't want to have a slew of static dependencies that would cause a deployment train every time a new change was introduced.
* We wanted deployability isolated with individual teams. We didn't want individual [Bounded Contexts](https://martinfowler.com/bliki/BoundedContext.html) to have to be gates for other Bounded Contexts to getting new features or iterations into production.
* As much as we could avoid it, we didn't want any top-level decisions affecting individual Micro Frontends. We wanted to avoid having to define a page-level version of React and forcing each Micro Frontend to conform, for example.
* We wanted to avoid heavy tooling that would require a lot of coordination and maintenance between teams, demanding time and attention from people where it could be better spent doing the actual work of building the product.

Web Components gave us almost all of these things by default, but as we approached it, we wanted to make a couple of recommendations to keep things as good as possible for our users. The first consideration was ensuring that our bundle sizes were as small as they could be.

### Building Micro Frontends

If you're using a client-side bundler like webpack, it's very easy to bundle third-party dependencies into your application code and get isolation within your bundle that way. In our case, when we have multiple individual Web Components on a single page, we wanted a way for each of these components to keep their bundle sizes low by loading third-party dependencies separately. If two Web Components rely on the same dependency, they would both load the dependency separately, and rely on browser caching to only load the dependency over the wire once.

This begets two problems, though, the first being that Web Components don't have enough JavaScript isolation to globally load two separate versions of one dependency.

#### Javascript Isolation

This is a glaring problem: Web Components are very isolated in every sense _except for in JavaScript_. The `window` and `document` in a Web Component are the same `window` and `document` in the surrounding page. So any third party dependency would either need to be externalized to the `window`, causing us to have to make a top-down decision about dependencies and versions that our Web Components can rely on, or we'd need some way to isolate these external dependencies without simply packaging them into our bundles.

We much preferred the second option, but couldn't find a way to do it, so we built one. Our teams are all using webpack for their build, so we wrote a webpack plugin to wrap the bundles in a closure, and include any [`externals`](https://webpack.js.org/configuration/externals/) within the closure so that our application can refer to them as global dependencies, without the need for those externals to actually be on the global context. The resulting package is called the [`isolated-externals-plugin`](https://github.com/WTW-IM/isolated-externals-plugin). Utilizing this plugin allows us to load a UMD package more than once on the same page without polluting the `window` object (so long as those UMD bundles refer to `this` and not the `window` directly).

There are trade-offs here, the primary one being that the resulting JavaScript heap will include these dependencies more than once, but remember — we'd run into that same problem if we were bundling our dependencies into our applications anyway. This way, we end up having those dependencies in the heap more than once, but we only need to load them over the wire a single time.

The second problem begotten by the effort to isolate our external dependencies is a particular one. What if one Web Component refers to an external dependency at one minor or patch version, and another refers to the same dependency at the same major version, but at a different minor or patch version? We'll end up transferring two packages over the wire, but probably unnecessarily.

#### Intelligently Utilizing Browser Caching

We'll have a very slow-loading page if we have many Web Components loading separate versions of external dependencies independently, but we can solve this problem in two steps.

1. Use [`unpkg`](https://unpkg.com/) as our "official" CDN to load these external dependencies.
2. Utilize `unpkg`'s [semver](https://semver.org/) range functionality to refer only to the major version of your dependencies. This way, even if one Web Component was built on React@16.13.0 and another one was built on React@16.13.1, they can both use [unpkg.com/react@16/umd/react.production.min.js](https://unpkg.com/react@16/umd/react.production.min.js) at runtime. With that url (`@16` being the key portion), both Components will load the latest version of React@16, which will include the functionality of the previous patch versions.

Doing these two things allows us to ensure that, generally, we'll only load a major version of a dependency over the wire once. Should teams see a need to be very specific about their versions, they can still specify an exact version, but for most cases, we'll be safe loading the versions of dependencies we need, and still limiting the bytes we transfer over the wire.

With a solution for caching our external dependencies in place, we could focus on a solution for caching and refreshing our bundles.

### File Naming, Cache-Busting, and Deployment

The traditional way of ensuring that browsers load the newest version of a javascript application is to append a new hash onto the filename with each deployment and set `Cache-Control` headers for these files to keep files for something like a year. For our case, we don't want downstream teams to know about a new hash because we don't want teams to need to be aware that a new version of a component was released. For our purposes, adding a hash to the filename would not work.

Instead, we decided to have our Components delivered with static file names (something like `shopping-component.js`), and we would rely on the [`ETag` Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag) and the built-in browser behaviors around it to manage cache busting.

To allow `ETag` headers to work, the server that delivers the static files simply needs to generate an `ETag` value for each new version of the file to compare against when the browser requests the static resource, and send that on the response with the content. When the browser sends a request for a file with an `ETag` Header value that matches the file on the server, the server returns a `304 Not Modified` response, and, the browser will fall back to its cached resource. When the `ETag` values are mismatched, the server will send the new version of the file along with a `200 OK` response.

In our case, most of our applications are built with [.NET Core](https://docs.microsoft.com/en-us/dotnet/core/). In that framework, the [`useStaticFiles` middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/static-files?view=aspnetcore-3.1#serve-files-inside-of-web-root) does the right thing with `ETag` headers out of the box. It will return a new `ETag` value for changed files, and will return a `304 Not Modified` response when the requested asset is unchanged.

There are tradeoffs with utilizing the `Etag` header to manage caching:

* The browser has to make a request to the server to confirm the `ETag` value.
* There isn't a one-size solution to utilize `Cache-Control` headers.

There are some mitigating factors there, though:

* The size of the 304 response is a few hundred bytes, which is typically minuscule to your actual bundle size (generally some tens-to-hundreds of kilobytes).
* If you're delivering a static file where you have a good understanding of the change rate (do we deploy new code every day, week, month, etc?), then you can set up `Cache-Control` headers to meet the needs of your specific case.

Understanding how we can deploy these files with a static name and consistent cache busting is really good, but in itself, it's not enough to give us the independence for teams that we're looking for. For that, teams need to be able to deploy their applications independently, without the need to notify other teams of a change, or to step on the toes of other teams by deploying through the same deployment pipeline.

#### Deploying Independently

There are a number of ways to solve the problem of isolated, independent deployments, but our solution is to utilize right-hand routing (configured in a load balancer) to determine which servers a particular request should go to.

In a traditional or monolithic application, traffic might move a little bit like this:

![Diagram describing all traffic from base-domain.com being sent to a single primary server, no matter the path that is being requested.](./monolithic-routing.png)

For our purposes, we use the path (the "right-hand route") to determine which of any number of servers to send traffic to:

![Diagram describing traffic from base-domain.com being sent to multiple different servers, depending on the path.](./right-hand-routing.png)

This allows our teams to manage their business domain (or Bounded Context) entirely, from how business logic is handled on the server side to how display and inputs are handled on the client side.

In order to deliver Web Components as single JavaScript files from multiple different servers, we simply deploy those files to these independent servers, and refer to them using our path-based routing. For example, a Shopping web component url may be something like `base-domain.com/shop/shopping-component.js`, which will be delivered by a server in the Shopping Bounded Context. For a separate Bounded Context, like Account, you might have a similar url for a Web Component — something like `base-domain.com/account/account-component.js` — which is delivered from a server in the Account Bounded Context.

With these pieces in place, a webpage can load different components from different servers, independently managed and deployed by disparate teams, with the confidence that new code will be properly cache-busted on the client side.

## Wrap-Up

All the considerations above made it possible for our teams to utilize Web Components as our primary means for delivering Micro Frontends that exist alongside each other on a single webpage, are built in React, can communicate with each other, and are deployed by different teams independently, and without the need for heavy-handed, top-down decisions or tooling. The ultimate result of all this is that teams can move quickly to iterate on ideas and features within their individual domains, and to steadily and easily improve our product over time for our users.

## Appendix A: About Browser Compatibility

With [Microsoft Edge moving to a Chromium base](https://support.microsoft.com/en-us/help/4501095/download-the-new-microsoft-edge-based-on-chromium), all major browsers now natively support Web Components. If you need to support older browsers, the [Web Components polyfills](https://www.webcomponents.org/polyfills) do an excellent job of providing backwards-compatibility.

We were having a very difficult time getting our components to function properly in pre-Chromium Edge, so we've added a banner to our site that appears for older versions of Edge (and IE11 and below) that encourages users to update to the new Edge (which will happen automatically for Edge-users anyway [with Windows updates](https://docs.microsoft.com/en-us/deployedge/microsoft-edge-sysupdate-windows-updates)), or to another evergreen browser.