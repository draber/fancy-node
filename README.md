# fancy-node
Access or create DOM nodes in style.

## Features
This package lets you conveniently access existing and create new DOM nodes. It also implements some interesting helper methods such as `waitFor()`. It works in a web browser as well as on the server-side where it uses [jsdom](https://www.npmjs.com/package/jsdom). 

## Installation
You can add `fancy-node` to your project with:
```bash
npm i fancy-node
```

## Usage
```javascript
// client-side
import fn from 'fancy-node'; 

// server-side
import fn from 'fancy-node/src/server.js';
// alternatively: const fn = require('fancy-node/src/server')
```
`fancy-node/src/server.js` sets up a `jsdom` environment but other than that it's identical to the client-side version.  

### Building elements
The basic syntax is `fn.<any>()` where `<any>` is the type of element you wish to create:
```javascript
const div1 = fn.div();
// -> <div></div>

const div2 = fn.div({options}); 
// -> <div id="foo" class="bar"></div>, we'll see about the options in a minute
```
You can use any native HTML or SVG element as a method, in fact, all function calls go through a `Proxy` that acts as a dispatcher. In other words, you could use `fn.span()`, `fn.ul()`, `fn.svg()` and so on. Web components are not supported, `fn.myComponent()` would fail.

#### Options
Options are a nested object with the following keys:
| Name         | Type    |Mandatory | Default     |
|:-------------|:--------|:-------- |:------------|
| `content`    | mixed   |no        | `undefined` |
| `attributes` | Object  |no        | `{}`        |
| `style`      | Object  |no        | `{}`        |
| `data`       | Object  |no        | `{}`        |
| `aria`       | Object  |no        | `{}`        |
| `events`     | Object  |no        | `{}`        |
| `classNames` | Array   |no        | `[]`        |
| `isSvg`      | Boolean |on SVG yes| `false`     |

##### `content`
This is, as you would have expected, the content of the element. It can be any of the following:
- a text string
- an HTML element with or without sub-elements
- a SVG element with or without sub-elements
- a piece of HTML code
- a piece of SVG code - this works only when the code starts with `<svg`!
- an array with any combination of the above
```javascript
// the examples assume fn.div({ content: ... }) as basis

content: 'some random text'
// -> <div>some random text</div>

content: fn.div({
    content: fn.span()
})
// -> <div><div><span></span></div></div>

content: '<div><span></span></div>'
// -> <div><div><span></span></div></div>

content: [
    'another random text',
    fn.span({
        attributes: {
            id: 'foo'
        },
        content: fn.strong({
            content: 'some text'
        })
    })
]
// -> <div>another random text<span id="foo"><strong>some text</strong></span></div>
```

##### `attributes`
```javascript
attributes: {
    id: 'foo',
    disabled: true // for boolean attributes
}
// -> <div id="foo" disabled>
```
While _attributes_ and _properties_ aren't the same thing `fn` combines them under `attributes`. Attributes and properties are accepted; if both are provided, _property_ takes precedence.

The following mapping lists the ambiguous cases: 

| Attribute         | Property          |
|:------------------|:------------------|
| `accesskey`       | `accessKey`       |
| `class`           | `className`       |
| `contenteditable` | `contentEditable` |
| `for`             | `htmlFor`         |
| `nomodule`        | `noModule`        |
| `tabindex`        | `tabIndex`        |
| `colspan`         | `colSpan`         |
| `rowspan`         | `rowSpan`         |

_Note: [jsdom doesn't support `contentEditable`](https://github.com/jsdom/jsdom/issues/1670) (and probably other properties). This is something you need to take into account when using this package to build HTML on the server!_

##### `style`
This accepts the same values you would set in `element.style`:
```javascript
style: {
    fontSize: '1.4rem', 
    border: '1px red solid'
}
// -> <div style="font-size: 1.4rem; border: 1px red solid">
```
You could also set a string in `attributes.style`; it will be merged into the `style` object. `style` takes precedence over `attributes.style`.

##### `data`
These values are applied to `element.dataset`.
```javascript
data: {
    foo: 'bar',
    bar: 42
}
// -> <div data-foo="bar" data-bar="42">
```
##### `aria`
Before you set anything ARIA-related consider the [first rule of ARIA use](https://www.w3.org/TR/using-aria/#firstrule): there probably already exists an [HTML element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element) for your particular purpose.

With that being said, you can set ARIA rules like this:
```javascript
aria: {
    role: 'button', // see above
    hidden: true, 
    label: 'Close'
}
// -> <div role="button" aria-hidden="true" aria-label="Close">
```
All rules but `role` will be prefixed with `aria-`.

##### `events`
Key-value pairs of events and their associated functions:
```javascript
events: {
    click: evt => {
        magic(evt.target);
    }
}
// -> div does something magical when clicked
```

##### `classNames`
An array of class names that will be added to `element.classList`. `attributes.className` or `attributes.class` are also supported. `classNames` and `attributes.className` or `attributes.class` will both be applied.

##### `isSvg`
This needs to be set to `true` for all SVG elements (`svg`, `path`, `circle`, etc.).

### Retrieving elements
Wrapping `document.querySelector()` and `document.querySelectorAll()` into `$()` or `$$()` is nothing new, the versions in `fancy-node` are borrowed from [Lea Verou](https://lea.verou.me/2015/04/jquery-considered-harmful/). 

The first argument for both functions `fn.$()` and `fn.$$()` is a CSS selector, the optional second argument is a scope element.

```javascript
const list = fn.$('ul');
// -> first <ul>

const firstLiElement = fn.$('li', list);
// -> first <li> element inside this <ul>

const allLiElements = fn.$$('li', list);
// -> all <li> elements inside this <ul>
```
### Other functions

#### `fn.toNode()`
This is used under the hood to power the `content` argument but is also publicly accessible. It accepts the same input as `content` and returns either an `HTMLElement`, a `TextNode` or a `DocumentFragment`, but either way a single node.
```javascript
fn.toNode('some random text')
// -> TextNode

fn.toNode(fn.div({
    content: fn.span()
}))
// -> HTMLElement '<div><span></span></div>'

fn.toNode('<div><span></span></div>')
// -> HTMLElement '<div><span></span></div>'

fn.toNode([
    'another random text',
    fn.span(
        attributes: {
            id: 'foo'
        }
    ),
    fn.svg({
        isSvg: true
    })
])
// -> DocumentFragment containing all of the above 
```

#### `fn.empty()`
This removes all content from an element without the shortcomings of `element.innerHTML`.
```javascript
const elem = fn.empty(fn.$('.bar'));
// -> convert <elem class="bar"><x><y>some</y></x><z>text</z></elem> to <elem class="bar"></elem>
```

#### `fn.waitFor()`
This waits for an element to be present in the DOM and takes the same arguments as `fn.$()`. It returns a Promise with the element as its value.
```javascript
const ul = fn.$('ul');
fn.waitFor('li.bar', ul) // scope is optional
    .then(element => {
        magic(element);
    })
// -> waits for `<li class="bar"> to be present before doing magic
```


## Build
The two reference files under `src/elements/listing` are built with [MDN Tag Collector](https://github.com/draber/mdn-tag-collector.git).
