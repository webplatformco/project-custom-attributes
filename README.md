# Custom Attributes

Authors: Lea Verou, Keith Cirkel

This document covers the history and motivation for custom attributes: use
cases, prior art, and the design questions raised along the way and how each
was resolved.

The current proposal is **[EXPLAINER.md](EXPLAINER.md)**.

1. [Introduction](#introduction)
2. [Use cases](#use-cases)
3. [Prior art](#prior-art)
   1. [Related proposals](#related-proposals)
   2. [Userland](#userland)
   3. [Other](#other)
4. [Design principles](#design-principles)
5. [Design decisions](#design-decisions)
   1. [How to specify?](#how-to-specify)
   2. [Reflection](#reflection)
   3. [Which `Attr` property stores the JS-facing value?](#which-attr-property-stores-the-js-facing-value)
   4. [API surface](#api-surface)
   5. [Scoping](#scoping)
   6. [Same attribute on multiple element types](#same-attribute-on-multiple-element-types)
   7. [Naming](#naming)
   8. [Lifecycle hooks](#lifecycle-hooks)
   9. [How to react to attribute changes?](#how-to-react-to-attribute-changes)
   10. [Traits involving multiple attributes](#traits-involving-multiple-attributes)
   11. [When does the constructor run?](#when-does-the-constructor-run)
6. [Current proposal](#current-proposal)
7. [FAQ](#faq)
   1. [Does this replace custom elements?](#does-this-replace-custom-elements)
   2. [Can't we do this already with `MutationObserver`?](#cant-we-do-this-already-with-mutationobserver)

## Introduction

A lot of reusable UI functionality is better expressed as composable traits or behaviors on existing elements, rather than whole new HTML elements (_has a_ rather than _is a_).

The platform already has this capability, through attributes.
Imagine if global attributes like `title`, `popover`, `lang`, `hidden` had to be implemented as elements.
Yet, this is the only tool web component authors have today.

Additionally, certain web components have an attribute counterpart to link them to another element (or link _to_ them from another element).
A native example here is `<input list>` and `<datalist>`. While `<datalist>` could be implemented as a web component, there is no authorland counterpart for the `list` attribute.

Besides the philosophical data modeling argument, overcomponentization introduces tangible problems.
Unlike framework components that can compile to much shallower DOM trees, inserting a whole new element in the DOM has a **cost**.
It affects **selector matching, DOM traversal, styling**, and many other things.

Inserting a custom element is not even allowed in all contexts.
Consider this:

```html
<sortable-table>
  <table>
    <thead>
      <!-- elided -->
    </thead>
    <tr>
      <td-value value="0.5">
        <td>Half</td>
      </td-value>
    </tr>
  </table>
</sortable-table>
```

Even when wrapping an element to add additional functionality is a viable solution, the ergonomics are considerably worse, with a significantly lower [signal-to-noise ratio](https://lea.verou.me/blog/2025/user-effort/#signal-to-noise). Compare:

```html
<dt-format type="relatve"><time datetime="2025-11-15"></time></dt-format>
```

with:

```html
<time datetime="2025-11-15" dt-format="relative"></time>
```

Being able to define custom attributes that can be used on any element also addresses several pain points around extending built-ins,
which was one of the most prominent pain points around Web Components per State of HTML 2025.
Authors can simply do `<button my-button>` rather than having to define their own `<my-button>` component that emulates or wraps buttons, introducing a ton of complexity.

## Use cases

A **[concrete list of use cases can be found here](use-cases.md)**.

## Prior art

### Related proposals

- [Minimal custom attributes extending `Attr`](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3597708609) by @keithamus
  - [Reflection via `attr.value`](https://github.com/WICG/webcomponents/issues/1029#issuecomment-2455166850)
- [Proposal: Custom attributes for all elements, enhancements for more complex use cases](https://github.com/WICG/webcomponents/issues/1029) by @leaverou
- [Custom Element Features, Built-in Enhancements, Itemscope Managers](https://github.com/WICG/webcomponents/issues/1000)
- [Original custom attributes proposal (naming only)](https://github.com/whatwg/html/issues/2271) by @leaverou
- [Element Behaviors](https://github.com/lume/element-behaviors) by @lume
- [@lume's custom attributes proposal](https://github.com/lume/custom-attributes) by @lume
- [Custom Attributes from the Web Components CG 2022 TPAC Report](https://w3c.github.io/webcomponents-cg/2022.html#custom-attributes) by @EisenbergEffect

### Userland

- [htmx](https://htmx.org/)
- [VueJS custom directives](https://vuejs.org/guide/reusability/custom-directives)
- [Angular directives](https://angular.dev/guide/directives/attribute-directives)
- [Svelte attachments](https://svelte.dev/docs/svelte/@attach)
- [SolidJS custom directives](https://www.solidjs.com/tutorial/bindings_directives)
- [Alpine.js custom directives](https://alpinejs.dev/advanced/extending)
- [Aurelia Custom Attributes ca2015](https://aurelia-1.gitbook.io/v1-docs/templates/custom-attributes)

### Other

- [Custom attributes for all elements (TPAC 2025 breakout)](https://github.com/w3c/tpac2025-breakouts/issues/46)

## Design principles

Generalizing the [PoC](https://www.w3.org/TR/design-principles/#priority-of-constituencies) as [consumers > producers](https://lea.verou.me/blog/2025/user-effort/#consumers-over-producers), we end up with this expanded PoC:

1. End-users
2. HTML authors
3. Custom attribute authors
4. Implementors
5. Spec authors
6. Philosophical purity

## Design decisions

> [!Important]
> These boxes are used for conclusions, based on the prose before them.
> Where a conclusion changed once the feature was specified and prototyped, the
> box records what we've currently settled on.

### How to specify?

The prevailing pattern seems to be defining a subclass of `Attr`.

This has several benefits:

- Existing API to use (e.g. `this.ownerElement` to refer to the host element)
- Existing mental model around "upgrading" nodes
- Because attribute nodes are accessible via `element.attributes`, this also provides a clash-free way to hang methods and other values.
- `Attr` is even an `EventTarget` so in theory attributes could even dispatch events

There are also some downsides:

- `Attr` is an old API, and comes with baggage. E.g. now we need to define how to handle namespaces too.

> [!Important]
> Custom attributes are subclasses of `Attr`.
> Namespaces are always HTML; a custom attribute always has a null namespace,
> and an attribute in any other namespace is never custom, even if its local
> name matches a definition.
> `new Attr()` itself throws a `TypeError`; only subclasses registered with a
> `CustomAttributeRegistry` can be constructed, mirroring `HTMLElement`.

### Reflection

The proposal that hosted most of the discussion proposed handling attribute-property reflection as well, as this is a big pain point when using WC APIs directly.

However, this opens this up to a lot of API design complexity and increases the API surface, while a custom attributes API can ship without it and still cover use cases.

> [!Important]
> Attribute-property reflection is deferred. The current proposal adds no
> reflection machinery; authors react to `attributeChangedCallback()` and store
> whatever they need on their `Attr` subclass.

### Which `Attr` property stores the JS-facing value?

Even if authors handle attribute-property reflection themselves, at the very least there needs to be a designated slot to hold the reflected value (which by default would be a string mirroring the attribute value).

Keith [proposed](https://github.com/WICG/webcomponents/issues/1029#issuecomment-2455166850) simply specifying accessors on `attr.value`:

```js
class extends Attr {
  get value() {
    return Number(super.value)
  }
  set value(value) {
    super.value = value;
  }
}
```

While elegant, this approach has several downsides:

- Internal consistency: No built-in attributes work that way. In fact, `Attr.prototype.value` is [defined](https://dom.spec.whatwg.org/#dom-attr-value) to be a string.
- In many cases there are very big differences between the JS-facing value and its string representation.
  For example, consider the `style` attribute and `element.style`, which is a whole object!
- Even when conversion is idempotent, we want to avoid any roundtrips that are not absolutely necessary, since these conversions are not always cheap.
- While `Attr` is not very widely used directly, being a very old API means there can be any number of scripts depending on `attr.value` being a string.

A platform-designated slot (`data`, `parsed`, etc) was considered, but it would need a default behaviour, a relationship to `value`, and a story for when the two disagree, none of which the use cases required.

> [!Important]
> No new property. `attr.value` stays a string and remains the source of truth
> for the attribute.
> Authors hold the parsed value in a field of their own choosing on the
> subclass and refresh it from `attributeChangedCallback()`.

### API surface

Many native features add methods etc to the element.
E.g. the `popover` attribute also adds `showPopover()`.

However, just like reflection, trying to specify this adds additional complexity, and is not strictly necessary:
with the model of `Attr` subclasses, authors can always hang methods on their `Attr` subclass, and they will be accessible via `element.getAttributeNode("attr-name").methodName()` or `element.attributes["attr-name"].methodName()`.

Authors can use additional JS features to improve ergonomics, such as [first-class protocols](https://github.com/tc39/proposal-first-class-protocols), [decorators](https://github.com/tc39/proposal-decorators),
or even monkey-patching, at their own risk.

An earlier draft floated a static `definedCallback(name, ElementConstructor)` hook so an attribute could react to being registered on a particular element class.
With registries scoped to trees rather than element classes (see [Scoping](#scoping)), there is no per-class registration to react to, so the hook has no job.

> [!Important]
> No primitive for adding element API, and no `definedCallback`.

### Scoping

Some proposals involve a global `customAttributes` registry, while in others `customAttributes` is a property of specific element classes, with `HTMLElement` serving as the global one.

Per-class registration is attractive because many use cases only make sense on certain element types, and the **same attribute** name may have entirely different meanings depending on the context (e.g. `for` is often used generically for element linking, and can mean completely different things).

It has real costs though:

- Looking up a definition for `(element, name)` has to walk the element's prototype chain, and must be redone whenever any registry on that chain changes.
- It only works for elements with a JS constructor in scope. Unknown elements, elements in other namespaces, and elements whose constructor is in another realm all need special-casing.
- It is a new scoping model, when the platform already has one for custom elements: `CustomElementRegistry` is scoped to documents, shadow roots, and elements.

As @annevk [points out](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3597830700):

> `CustomElementRegistry` can be scoped to documents, shadow roots, and **elements**. And `document.customElementRegistry` is probably what we want to mimic for anything new. Not sure we should add another global accessor for this.

@sorvell also [talked](https://github.com/WICG/webcomponents/issues/1029#issuecomment-1718332785) about scoped registries:

> Experience with customElements and the scoped registries proposal suggests that scoping is a must and to avoid the pain custom elements has gone through, this feature shouldn't ship without it.

Reusing the custom element scoping model gets tree-scoped registries essentially for free, and sidesteps the per-class lookup problem entirely.

> [!Important]
> Registries are scoped to **trees**, not element classes, mirroring
> `CustomElementRegistry` exactly:
>
> - `window.customAttributes` is the document's global `CustomAttributeRegistry`.
> - `new CustomAttributeRegistry()` creates a scoped registry, attached to a
>   subtree via `registry.initialize(root)`,
>   `attachShadow({ customAttributeRegistry })`,
>   `createElement(name, { customAttributeRegistry })`,
>   `importNode(node, { customAttributeRegistry })`, or
>   `setHTML()`/`setHTMLUnsafe()` options.
> - Every `Element`, `ShadowRoot` and `Document` has a `customAttributeRegistry`
>   (null or a registry); once set it cannot change.
> - A definition applies to **any** element in the registry's scope, including
>   SVG and MathML elements. Restricting to certain element types is the author's
>   job (see [Specialising a built-in](EXPLAINER.md#specialising-a-built-in)).

### Same attribute on multiple element types

There are many use cases where **the same attribute needs to apply to multiple element types**, without it being global.
Examples abound in the platform: `href`, `src`, several form control attributes, loading attributes like `loading` or `crossorigin`, etc.

With tree-scoped registries this is the default: one `define()` call covers every element in the tree.

E.g. a `persist-value` attribute that persists form control values in localStorage:

```js
// persist-value.js
export class PersistAttr extends Attr {
  connectedCallback() {
    if (!("value" in this.ownerElement)) return;
    // elided
  }
}

customAttributes.define("persist-value", PersistAttr);
```

Custom form controls get it too, as long as they expose whatever `PersistAttr` needs (here a `value` property).

> [!Important]
> Nothing to specify. One definition applies to all elements in scope; authors
> opt out per element type in their own code.

### Naming

Originally, **hyphens** were suggested, as a way to mirror custom element naming rules.
However, there are many exceptions in the web platform making this a bit awkward, e.g.:

- A ton of SVG presentation attributes (e.g. `fill-opacity`)
- `aria-*`
- `data-*`
- `allow-charset`
- `http-equiv`

Another issue making this hard is that many custom element attributes use hyphens.
On two different social media polls, about half of authors voted that they prioritize readability over [platform consistency](https://www.w3.org/TR/design-principles/#naming-consistency) (which recommends concatcase):

- <https://x.com/LeaVerou/status/1863812496106389819>
- <https://front-end.social/@leaverou/113587139842885936>

<!--
Additionally, as @jakearchibald [pointed out](https://github.com/WICG/webcomponents/issues/1029#issuecomment-2454991200), even attributes without dashes often camelCase in JS, which would make reflection clash (e.g. `readonly` → `readOnly`).
Of course that is not a problem if we don't handle reflection automatically.
-->

Another option would be a specific **prefix**, though given that the attribute name itself often needs to be namespaced with a library prefix, this would only produce acceptable ergonomics if very short, e.g. a single character (`#`, `$`, `:`, `@` etc).

The CSS custom ident prefix (`--`) has also been proposed.
On one hand it is the target of numerous author complaints, on the other it _is_ an existing established convention.

Since definitions apply to every element (including SVG and MathML), the excluded set has to cover the whole platform, not just HTML.
The hyphenated SVG presentation attributes turn out not to be a problem: they are only meaningful on SVG elements, and an author defining `fill-opacity` as a custom attribute gets exactly what they asked for.
What must be excluded is names with platform-wide meaning that can appear on any element.

> [!Important]
> A hyphen is required, as with custom elements. HTML, SVG and MathML avoid
> adding hyphenated attribute names going forward, so this is the
> forward-compatibility guarantee.
> Names must start with a lowercase ASCII letter and contain no uppercase
> letters, so HTML's case-insensitive attribute handling always resolves.
> Reserved: anything starting with `aria-`, `data-`, `xml` or `xlink`, plus
> `accept-charset` and `http-equiv`. `data-*` is excluded because it already
> has platform semantics (`dataset`), even though it is authorland.

### Lifecycle hooks

What does `connectedCallback` etc mean in the context of an attribute?
Do they still correspond to the _element_ being connected, or the _attribute_ being specified on the element? Or when both are true?

@DeepDoge [makes a good case](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3599188181) for the latter:

> IMO custom attributes are composable behavior units, kind of like a superset of extended custom elements. So, `connectedCallback()` should run only when both are true:
>
> - The attribute is attached to an element
> - That element is connected to the DOM
>
> If we simplify it even more, it should trigger when the attribute is connected to the DOM, not when attribute is connected to an element.
>
> Just like how a custom element's `connectedCallback()` gets triggered when the element is actually connected to the DOM, not when it has a parent.
>
> The whole point of `connectedCallback()` / `disconnectedCallback()` is to initialize or clean things up depending on whether the thing (element or attribute) is live in the DOM.
>
> So should it be called "when the attribute is connected, or when the element is connected?": It should be called when the attribute is connected to the DOM, which also requires element its connect to be connected to the DOM as well. I think we can all agree that "Connected" means "Connected to the DOM".

This generalizes cleanly to the rest of the custom element callbacks, because an attribute is only ever "in the DOM" through its element:

- `adoptedCallback` follows the element into the new document.
- `connectedMoveCallback` fires when the element is moved with `moveBefore()`, with the same disconnected-then-connected fallback as custom elements.
- Moving an `Attr` node between elements is impossible without removing it first (`setAttributeNode()` on a node that already has an owner throws), so there is no separate "attribute adopted" notion.

No use case in [use-cases.md](use-cases.md) needed element lifecycle separate from attribute lifecycle.

> [!Important]
> An attribute is _connected_ when it is in an element's attribute list **and**
> that element is connected. `connectedCallback`, `disconnectedCallback`,
> `connectedMoveCallback` and `adoptedCallback` all key off that, whether the
> transition is caused by the attribute being added/removed or by the element
> being inserted/removed/moved/adopted.
> Element-specific hooks have been deferred.

### How to react to attribute changes?

While it may seem at first that simply specifying a setter on `attr.value` would help us react to attribute changes, that is not what actually happens today ([demo](https://codepen.io/leaverou/pen/azNQJdP?editors=1012)).
The getter of `attr.value` simply returns the value of an internal slot, so setting attribute values does not go through it at all.

While a mutation observer is always an option, reusing `attributeChangedCallback()` seems like a very fitting solution, and on par with reusing existing lifecycle hooks.

> [!Important]
> `attributeChangedCallback(oldValue, newValue)` fires when the attribute is
> added, changed, replaced or removed, with a similar signature as custom
> elements. `oldValue` is null on add and `newValue` is null on remove. As the
> attribute name is always consistent, and namespace will always be `null`,
> these arguments have been dropped.
> On upgrade it also fires once with the initial value, before
> `connectedCallback`, so an attribute never has to special-case its first value.

### Traits involving multiple attributes

While not MVP, there are many use cases where a feature utilizes multiple attributes.
A common pattern is when one attribute enables the feature, and the rest customize it.

For example, in the web platform there is `<template shadowrootmode="open">`, but also several `shadowroot*` attributes that provide parameters (`shadowrootdelegatesfocus` etc).

Another pattern is where multiple attributes work together to specify a DSL. For example [Vue directives](https://vuejs.org/api/built-in-directives.html) (`v-if`, `v-for`, `v-on` etc).

Reusing `observedAttributes` to let an attribute watch _other_ attributes on its element was considered.
It was left out of v1 for two reasons:

- Cost: today an attribute change only enqueues a reaction if that attribute is itself custom, so non-custom attribute mutations stay on the fast path. Observing arbitrary attributes would require every custom attribute on an element to be consulted on every attribute change.
- Ambiguity: `attributeChangedCallback` would then receive names other than the attribute's own, and an attribute reacting to a sibling's change is one `MutationObserver` on `this.ownerElement` away (see [Reacting to sibling attributes](EXPLAINER.md#reacting-to-sibling-attributes)).

> [!Important]
> No `observedAttributes` in v1. `attributeChangedCallback` only fires for the
> attribute itself. Can be added later without breaking anything.

### When does the constructor run?

Native `Attr` nodes are constructed as part of parsing or `setAttribute()`, before any script involved in creating the element gets to run ([demo](https://codepen.io/leaverou/pen/azNQWoz?editors=1012)).

Running author code at that moment is unappealing:

- Attributes are created in bulk by the parser, which cannot run script between tokens.
- A custom element's constructor has not run yet, so an attribute registered for it cannot rely on the element being initialized.
- Most attributes are created on disconnected elements (fragments, templates, `createElement()`) that may never be inserted.

Custom elements solve exactly this with **upgrades**: the node is created as a plain node and converted in place, as a reaction, once it is connected. The same model fits attributes with one adjustment: an attribute on a disconnected element is not upgraded until the element is connected (or `upgrade()` is called explicitly), because most disconnected attributes are transient.

> [!Important]
> An `Attr` node is upgraded (its prototype swapped, its constructor run) as a
> custom element reaction, when:
>
> - its element becomes connected;
> - it is added to an already-connected element;
> - `define()` is called and it already exists on a connected element;
> - `registry.upgrade(root)` or `registry.initialize(root)` is called, connected
>   or not.
>
> The constructor runs with `this.ownerElement` and `this.value` already
> populated (the `super()` call returns the existing node), and must return
> `this`. Node identity is preserved: the same `Attr` object is upgraded in
> place.
> For an element that is both a custom element and carries custom attributes,
> the element's own reactions are enqueued before its attributes' reactions.
> `document.createAttribute(name)` is the one synchronous path: if `name` is
> defined in the document's registry, the constructor runs immediately.

## Current proposal

The design decisions above are consolidated into **[EXPLAINER.md](EXPLAINER.md)**: API, key scenarios, detailed design, WebIDL, and the list of ideas deferred to a later iteration.

## FAQ

### Does this replace custom elements?

No, they have distinct purposes, just like attributes and elements have distinct purposes in the platform.
Some things are better suited to attributes, and others to elements.

For example, you wouldn't want to implement a text field by doing `<div my-textfield textfield-value="foo" textfield-autofocus></div>`.
Ew!
An element is or isn't a text field, it's not something you can just slap on any element.

That said, _specializing_ an element type is a totally valid use case. E.g. `<input type="password" pwd-toggle>` or even `<button my-button>`.
For more background/motivation, check out the [Introduction](#introduction).

### Can't we do this already with `MutationObserver`?

Not really.

First, currently the only allowable namespace for custom attributes per spec is still `data-*`.
Coupled together with a library’s own prefix, this makes every attribute comically verbose.

`MutationObserver` does not work across shadow roots (though there is an [open issue](https://github.com/whatwg/dom/issues/1287) for observing open roots).
Even if it were, there is no way to run preparatory code before the element is connected.

`MutationObserver` is for reacting to future changes.
To react to existing uses of the attribute, we'd also need [`querySelectorAll()` improvements](https://github.com/whatwg/dom/issues/1422).

But even if all the moving pieces were there, having a primitive for this makes it easier to document, type, explain, and distribute.

A similar argument could have been made for custom elements: All the moving pieces were similarly there, but there was still value in being able to package the functionality up.
