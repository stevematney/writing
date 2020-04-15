# The Big Guide to Web Components Built With React

Our team at WTW wanted to find a way to deploy multiple small applications to the same page in a way that allows teams to separately develop, build, and deploy their apps in isolation, and without having to rely on other people or teams in order to get their work into a production environment. (This approach is often called [Micro Frontends](https://micro-frontends.org/).)

We researched a lot of different way of handling this problem, but finally landed on utilizing Web Components as our solution. This allows us to deploy individual, isolated Javascript applications together on a single page without much responsibility on other teams to expend a lot of work to integrate those components.

The primary web technology our shop uses is React. Using React with Web Components was a primary need for us. We've learned a lot as we've treaded the path of building isolated Web Components with React.

## What are Web Components?

[MDN has a very thorough and clear guide on Web Components](https://developer.mozilla.org/en-US/docs/Web/Web_Components), but in the simplest terms, Web Components is a way to build encapsulated elements rendered in isolation from the other elements on the DOM. This isolation allows you to have modularity and portability in individual client-side components, indifferent to their surrounding context.

This portability and isolation can enable many of the benefits of Micro Frontends, which is why we decided to pursue Web Components as a Micro Frontends solution, which [we'll talk about later](#web-components-as-micro-frontends).

## Why Build Web Components With React? Why not just _use React_?

In a Web Component, you're not limited in the technology you use, so any approach should be feasible, but if you're using a technology like React already, why not simply ship shared React components for different pieces of functionality?

There are at a few answers to this question: 

1. Shared components within the React system depend on a common set of React functionality. If you're building a component, and you want to use a newer feature, but the global version of React in your system does not support this feature, then you simply can't use it. A Web Component has no need to have a top-down version of React dictating the features it can use. The Web Component can use the version it needs.

2. Evolvability: the isolation of Web Components allows to you use React or really any other technology you see fit. If your team decides that something better or more fitting has come along for your Web Component, you can change it without worrying about affecting the surrounding page. Or you can build a Web Component without any third party library at all, and it can integrate in any web page easily (even those built with React).

3. Possibly the most important answer is deployability. One of the main concerns that a Micro Frontends approach aims to alleviate is a lack of dependency on external teams when deploying an application. If you're using React to piece together your frontend, all of your components need to be up to date at build time. With a Web Component whose source is hosted on an independent server, you can deploy new source code (which we [go into detail about](#deployment-and-routing) below), and your downstream dependents will be updated automatically.

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

The solution to this problem is to retarget events to the Shadow DOM, rather than to the original root node of the mounted element (`document.body`). [This React issue](https://github.com/facebook/react/issues/9242) describes a few different ways to solve this problem, most notably [`react-shadow-dom-retarget-events`](https://github.com/spring-media/react-shadow-dom-retarget-events), which works by adding an event listener for react-specific events to the Shadow DOM. This method works, to an extent, but can potentially be brittle (like if React adds new events).

Our approach has been a little bit different: we wrote a Javascript Class called [`react-html-element`](https://github.com/WTW-IM/react-html-element) that extends `HTMLElement`, creates a Shadow DOM, adds the appropriate `createElement` functions to it,  and sets the Shadow DOM as the `ownerDocument` for the `mountPoint` where your React app is intended to be mounted. This solves the event retargeting problem in a robust way because it doesn't depend on any specific React implementation details, and simply allows the Shadow DOM to capture all events from within the React application. You can see the [same example from above, but using `ReactHTMLElement` ](https://codesandbox.io/s/react-web-component-with-reacthtmlelement-7gsrj?file=/src/index.js), and now the functionality is working. (Displayed below.)

<iframe
     src="https://codesandbox.io/embed/react-web-component-with-reacthtmlelement-7gsrj?fontsize=14&hidenavigation=1&initialpath=%2Fhome&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="React Web Component With ReactHTMLElement"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

This was our first consideration, as it is the most clear problem when building Web Components with React. The next problem we wanted to solve was: how do we render HTML Children within a Web Component?

## Rendering Externally-Provided Children

The Web Components APIs provide [a very simple way of rendering children using `<slot>` tags](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_templates_and_slots#Adding_flexibility_with_slots). Luckily, we found that using `<slot>` in React was easy, and it just works. You can manage `<slot>` tags with names, but the absolute simplest way is just to include a bare `<slot>` tag in your React component:

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

Eventing is a [robust way of transferring complex data objects](https://developer.mozilla.org/en-US/docs/Web/Guide/Events/Creating_and_triggering_events), the implementation of a listener or dispatcher can be close to the point where that event information is  utilized,  your components can dispatch without knowing what might be listening, and they can listen without knowing what might dispatch.

## Styles

### Styled Components

### CSS Modules with Webpack

## React Portals and document.body

## Web Components as Micro Frontends

### Building (Isolating Javascript and Caching)

### Deployment and Routing

### Caching React Versions

