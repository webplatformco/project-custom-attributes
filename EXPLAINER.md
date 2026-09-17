# Custom Attributes

Authors: Lea Verou, Keith Cirkel

Motivation, use cases, prior art and the reasoning behind each decision are in
[README.md].

The TL;DR is "Custom Elements, But Attributes". Almost all semantics are copied
from Custom Elements, where possible.

1. [Proposed API](#proposed-api)
   1. [Defining a custom attribute](#defining-a-custom-attribute)
   2. [Lifecycle callbacks](#lifecycle-callbacks)
   3. [Registries](#registries)
   4. [Valid names](#valid-names)
2. [Key scenarios](#key-scenarios)
   1. [Behaviour on any element](#behaviour-on-any-element)
   2. [Specialising a built-in](#specialising-a-built-in)
   3. [Attributes on custom elements](#attributes-on-custom-elements)
   4. [Attribute with methods](#attribute-with-methods)
   5. [Scoped registry in a shadow tree](#scoped-registry-in-a-shadow-tree)
   6. [Reacting to sibling attributes](#reacting-to-sibling-attributes)
3. [Detailed design](#detailed-design)
   1. [`Attr` subclasses](#attr-subclasses)
   2. [Upgrades](#upgrades)
   3. [Reaction timing and ordering](#reaction-timing-and-ordering)
   4. [Registry lookup and scoping rules](#registry-lookup-and-scoping-rules)
   5. [Failure](#failure)
   6. [WebIDL](#webidl)
4. [Future work](#future-work)

## Proposed API

Essentially a `customAttributes` global is analogous to `customElements`; a
`CustomAttributeRegistry` which can define attributes, attached to elements:

### Defining a custom attribute

```js
class PersistValue extends Attr {
  constructor() {
    super();
    // this.ownerElement and this.value are available when upgraded from markup
  }

  connectedCallback() {
    let stored = localStorage.getItem(this.key);
    if (stored !== null) this.ownerElement.value = stored;
    this.ownerElement.addEventListener("input", this);
  }

  disconnectedCallback() {
    this.ownerElement.removeEventListener("input", this);
  }

  attributeChangedCallback(oldValue, newValue) {
    this.key = newValue || this.ownerElement.name;
  }

  handleEvent() {
    localStorage.setItem(this.key, this.ownerElement.value);
  }
}

customAttributes.define("persist-value", PersistValue);
```

```html
<input name="email" persist-value />
<textarea name="bio" persist-value="draft-bio"></textarea>
```

Every `persist-value` attribute on a connected element is upgraded to a
`PersistValue` instance. `el.getAttributeNode("persist-value")` and
`el.attributes["persist-value"]` return that instance.

### Lifecycle callbacks

An attribute is _connected_ when it is in an element's attribute list **and**
that element is connected.

| Callback                                       | Arguments                                         | Runs when                                                                                        |
| ---------------------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `constructor()`                                | none                                              | `document.createAttribute()` is called with a defined name, or the class is constructed directly |
| `attributeChangedCallback(oldValue, newValue)` | null `oldValue` on add, null `newValue` on remove | the value is added, changed, replaced or removed; once on upgrade with its initial value         |
| `connectedCallback()`                          | none                                              | added to a connected element, or its element is inserted into a document                         |
| `disconnectedCallback()`                       | none                                              | removed from a connected element, or its element is removed                                      |
| `connectedMoveCallback()`                      | none                                              | its element is moved while connected via `moveBefore()`                                          |
| `adoptedCallback(oldDocument, newDocument)`    | documents                                         | its element is adopted into another document                                                     |

Only the custom attribute's own changes fire `attributeChangedCallback`.
Callbacks are read from `constructor.prototype` once, at `define()` time.

### Registries

```js
// Global registry for the document, mirrors window.customElements
customAttributes.define("my-tooltip", MyTooltip);

// Scoped registry, mirrors scoped CustomElementRegistry
let registry = new CustomAttributeRegistry();
registry.define("my-tooltip", OtherTooltip);

let shadow = host.attachShadow({
  mode: "open",
  customAttributeRegistry: registry,
});
let el = document.createElement("div", { customAttributeRegistry: registry });
let copy = document.importNode(template.content, {
  customAttributeRegistry: registry,
});
el.setHTMLUnsafe(html, { customAttributeRegistry: registry });
registry.initialize(existingSubtree);
```

- `window.customAttributes` is the document's global `CustomAttributeRegistry`.
- `Element`, `ShadowRoot` and `Document` expose `customAttributeRegistry` (null
  or a registry). It is fixed at creation and cannot be changed.
- `define()`, `get()`, `getName()`, `whenDefined()`, `upgrade()` and
  `initialize()` have the same shape and semantics as on `CustomElementRegistry`.
- `upgrade(root)` upgrades attributes in disconnected subtrees;
  `initialize(root)` additionally assigns a scoped registry to nodes that have
  none.

### Valid names

A custom attribute name must:

- be a valid attribute local name;
- start with an ASCII lowercase letter and contain no ASCII uppercase letters;
- contain a `-`;
- not start with `aria-`, `data-`, `xml` or `xlink`;
- not be `accept-charset` or `http-equiv`.

`define()` throws `SyntaxError` otherwise. HTML, SVG and MathML avoid adding
hyphenated attribute names going forward, so hyphenated names are safe for
authors.

## Key scenarios

### Behaviour on any element

```html
<p loading-placeholder="3 sentences"></p>
<time datetime="2025-11-15" dt-format="relative"></time>
<svg><circle r="10" my-tooltip="Centre"></circle></svg>
```

One `define()` covers HTML, SVG and MathML elements alike.

### Specialising a built-in

```html
<input type="password" pwd-toggle />
<table sortable-cols>
  <button href="/save">Save</button>
</table>
```

Replaces most uses of customized built-ins (`is=""`).
The attribute checks `this.ownerElement` and no-ops or warns where it does not
apply:

```js
class PwdToggle extends Attr {
  connectedCallback() {
    let el = this.ownerElement;
    if (!(el instanceof HTMLInputElement) || el.type !== "password") return;
    // elided
  }
}
```

### Attributes on custom elements

```html
<range-slider persist-value with-tooltip></range-slider>
```

The element's own custom element reactions run before its attributes' reactions,
so an attribute's `connectedCallback()` sees an upgraded host.

### Attribute with methods

```js
class Removable extends Attr {
  remove() {
    this.ownerElement.dispatchEvent(new Event("remove"));
    this.ownerElement.remove();
  }
}
customAttributes.define("removable", Removable);

el.getAttributeNode("removable").remove();
```

Methods live on the attribute node, so they never clash with element API.

### Scoped registry in a shadow tree

```js
class Card extends HTMLElement {
  static registry = new CustomAttributeRegistry();
  static {
    Card.registry.define("card-action", CardAction);
  }

  constructor() {
    super();
    this.attachShadow({ mode: "open", customAttributeRegistry: Card.registry });
  }
}
```

`card-action` inside the shadow tree resolves to `CardAction` regardless of what
the page defines under the same name.

### Reacting to sibling attributes

```js
class Format extends Attr {
  #observer = new MutationObserver(() => this.#render());

  connectedCallback() {
    this.#observer.observe(this.ownerElement, {
      attributes: true,
      attributeFilter: ["datetime"],
    });
    this.#render();
  }

  disconnectedCallback() {
    this.#observer.disconnect();
  }

  #render() {
    /* elided */
  }
}
customAttributes.define("dt-format", Format);
```

## Detailed design

### `Attr` subclasses

- A custom attribute is an `Attr` whose _custom attribute state_ is `custom`.
  Every attribute has a state (`undefined`, `failed`, `precustomized`, `custom`)
  and a definition, mirroring elements.
- Custom attributes always have a null namespace. Namespaced attributes are
  never custom even if the local name matches.
- `Attr`'s constructor is annotated `[CustomAttrConstructor]`, the analogue of
  `[HTMLConstructor]`:
  - `new Attr()` throws `TypeError`.
  - `new MyAttr()` throws `TypeError` unless `MyAttr` is defined in the
    document's registry (or in the registry currently upgrading it).
  - Outside an upgrade it returns a new detached `Attr` named after the
    definition, null namespace, empty value, state `custom`. No callbacks run
    until it is attached with `setAttributeNode()`.
  - `ownerElement` and `value` are readable in the constructor, after `super()`.
- `document.createAttribute(name)` runs the constructor synchronously if `name`
  is defined in the document's registry.

### Upgrades

Attributes are created as plain `Attr` nodes (by the parser, `setAttribute()`,
or cloning) and upgraded in place as a custom element reaction.

_Try to upgrade_ an attribute is invoked from:

- the DOM insertion steps, for every attribute of every inserted element whose
  parent is connected;
- the append and replace attribute steps, when the element is connected;
- `define()`, for every matching attribute on connected elements in the
  registry's scope, in shadow-including tree order;
- `upgrade()` and `initialize()`, regardless of connectedness.

Attributes on disconnected elements are deliberately left alone until
connection.

_Upgrade an attribute_, given a definition (largely the same as Custom Elements):

1. Return if state is not `undefined`.
2. Set the definition; set state to `failed` (guards re-entrancy).
3. Enqueue `attributeChangedCallback(null, value)`.
4. If the element is connected, enqueue `connectedCallback()`.
5. Push the attribute on the definition's construction stack, set state to
   `precustomized`, construct with no arguments.
6. If construction throws or returns anything other than the attribute, see
   [Failure](#failure).
7. Set state to `custom`.

### Reaction timing and ordering

- Attributes join the existing custom element reactions stack; `[CEReactions]`
  is on `define()`, `upgrade()`, `initialize()` and `createAttribute()`.
- Timing is identical to custom elements: reactions run at the end of the
  outermost `[CEReactions]` scope, or at the next microtask checkpoint for
  parser-created nodes.
- Within one operation: `attributeChangedCallback` precedes `connectedCallback`
  or `disconnectedCallback`.
- On insertion, for each element in tree order: the element's own custom element
  reactions run, then its attributes' reactions in attribute-list order.
- `innerHTML` on a connected element upgrades before the `[CEReactions]` scope
  returns; `<template>` content, `DOMParser` documents and fragments upgrade
  when inserted into a connected tree.

### Registry lookup and scoping rules

- A document is created with a fresh global registry (`window.customAttributes`).
- `new CustomAttributeRegistry()` is _scoped_. Passing a non-scoped registry
  other than the document's own to `createElement`, `importNode`,
  `attachShadow`, `setHTML`, `setHTMLUnsafe` or `initialize()` throws
  `NotSupportedError`.
- An element's registry is set at creation from the options, else from the
  parent tree (shadow root, then document), and never changes.
- Cloning and adoption carry a scoped registry with the node; a non-scoped
  registry is replaced by the destination document's global registry.
- Lookup for `(element, attribute)`: the element's registry, then the definition
  whose name equals the attribute's local name, if the attribute's namespace
  is null.
- `define()` validation order: `TypeError` if not a constructor; `SyntaxError`
  on invalid name; `NotSupportedError` if the name or constructor is already
  defined in this registry or `define()` is re-entered.

### Failure

If the constructor throws or returns a different object during upgrade, the
definition is cleared, the state stays `failed`, queued reactions for that
attribute are dropped, and the exception is reported (rethrown for
`createAttribute()`). The node is never upgraded again.

Prefer no-op plus `console.warn()` over throwing to reject unsupported host
elements.

### WebIDL

```webidl
[Exposed=Window]
interface CustomAttributeRegistry {
  constructor();

  [CEReactions] undefined define(DOMString name, CustomAttributeConstructor constructor);
  (CustomAttributeConstructor or undefined) get(DOMString name);
  DOMString? getName(CustomAttributeConstructor constructor);
  Promise<CustomAttributeConstructor> whenDefined(DOMString name);
  [CEReactions] undefined upgrade(Node root);
  [CEReactions] undefined initialize(Node root);
};

callback CustomAttributeConstructor = Attr ();

partial interface Window {
  readonly attribute CustomAttributeRegistry customAttributes;
};

partial interface Attr {
  [CustomAttrConstructor] constructor();
};

partial interface mixin DocumentOrShadowRoot {
  readonly attribute CustomAttributeRegistry? customAttributeRegistry;
};

partial interface Element {
  readonly attribute CustomAttributeRegistry? customAttributeRegistry;
};

partial dictionary ElementCreationOptions {
  CustomAttributeRegistry? customAttributeRegistry;
};

partial dictionary ImportNodeOptions {
  CustomAttributeRegistry customAttributeRegistry;
};

partial dictionary ShadowRootInit {
  CustomAttributeRegistry? customAttributeRegistry;
};

partial dictionary SetHTMLOptions {
  CustomAttributeRegistry customAttributeRegistry;
};

partial dictionary SetHTMLUnsafeOptions {
  CustomAttributeRegistry customAttributeRegistry;
};

partial interface Document {
  [CEReactions, NewObject] Attr createAttribute(DOMString localName);
};
```

## Future work

The following items were considered, but descoped in order to get something
usable for a first iteration. It is believed the first iteration does not close
off the design space for these, so they are possible future extensions.

### Attribute-property reflection

A declarative mapping between `value` and a typed property, as
[WICG/webcomponents#1029] originally proposed.

Any design here should also be shared with custom elements as most consumers
will interact with the _element_, not the _attribute node_. Solving it for
attributes alone would create two reflection systems. Any work for both can be
done _after_ release of Custom Attributes, as a separate proposal.

### Parsed value slot

A platform-designated property (`data`, `parsed`) holding the converted value,
defaulting to `value`.

Overriding the `value` property is possible but may cause issues if downstream
code expects a String. Attributes can provide their own computed properties
in the subclass, similar in design to e.g. input's `valueAsDate`, and refresh it
from `attributeChangedCallback()`.

### Observing other attributes

`static observedAttributes` on the `Attr` subclass, feeding sibling changes
into `attributeChangedCallback()`.

Cost concern: every attribute change on an element would have to consult each
custom attribute on it. A cheaper shape may be to reuse whatever lands for
[observing connectedness] or a `MutationObserver` that can be attached from a
lifecycle callback without teardown boilerplate.

### Adding API to the host element

`popover` adds `showPopover()` to elements; authors may want the same.

There is no design for this in this proposal, but methods can be added to the
subclass and elements delegate (`el.attributes["attr-name"].method()`).
Meanwhile other platform solutions may answer this: decorators, first-class
protocols on the userland side, or a [platform provided behaviours] once those
settle.

### Per-element-class registration

`HTMLInputElement.customAttributes.define()` style scoping, and with it a static
`definedCallback(name, ElementConstructor)`.

This was rejected due to the cost of prototype-chain lookup and the additional
scoping model. The Scoped Registry model for Custom Elements feels worthy for
encapsulating sub trees, allowing libraries to avoid attributes being clobbered.

With a Scoped Registry model is could be possible to experiment with a
per-element scoped model in userland, and so if demand materialises, it could be
layered as a filter on top of tree-scoped registries (e.g. a `define()` option
listing element constructors) rather than as a separate registry per class.

### Namespaced custom attributes

Providing hooks into foreign elements or namespaces, or using custom namespaces
as a surface area.

This proposal ignores namespaces - custom attributes are always have a `null`
namespace due to their integration with HTML.

Nothing stops this being added later, but the use cases did not seem strong
enough.

[README.md]: README.md
[WICG/webcomponents#1029]: https://github.com/WICG/webcomponents/issues/1029
[observing connectedness]: https://github.com/whatwg/dom/issues/533
[platform provided behaviours]: https://github.com/whatwg/html/issues/12150
