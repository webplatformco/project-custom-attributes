# Custom Attributes

Authors: Lea Verou, Keith Cirkel

1. [Introduction](#introduction)
2. [Prior art](#prior-art)
	1. [Related proposals](#related-proposals)
	2. [Userland](#userland)
	3. [Other](#other)
3. [Design principles](#design-principles)
4. [Design decisions](#design-decisions)
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
5. [Current Proposal](#current-proposal)
	1. [Summary](#summary)
	2. [New members on existing interfaces](#new-members-on-existing-interfaces)
	3. [`Attr` subclass members](#attr-subclass-members)
	4. [Lifecycle hooks](#lifecycle-hooks-1)
6. [Notes / Patterns](#notes--patterns)
	1. [Persistent attribute node](#persistent-attribute-node)
	2. [Timing](#timing)
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
	<thead><!-- elided --></thead>
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

### How to specify?

The prevailing pattern seems to be defining a subclass of `Attr`.

This has several benefits:
- Existing API to use (e.g. `this.ownerNode` to refer to the host element)
- Existing mental model around "upgrading" nodes
- Because attribute nodes are accessible via `element.attributes`, this also provides a clash-free way to hang methods and other values.
- `Attr` is even an `EventTarget` so in theory attributes could even dispatch events

There are also some downsides:
- `Attr` is an old API, and comes with baggage. E.g. now we need to define how to handle namespaces too.

> [!Important]
> Despite drawbacks, extending `Attr` seems like a very internally consistent solution and solves many problems.

### Reflection

The proposal that hosted most of the discussion proposed handling attribute-property reflection as well, as this is a big pain point when using WC APIs directly.

However, this opens this up to a lot of API design complexity and increases the API surface, while a custom attributes API can ship without it and still cover use cases.

> [!Important]
> Let’s defer handling attribute-property reflection and provide sufficient low-level primitives to allow authors to make their own decisions.

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

> [!Important]
> Let’s use a separate property (e.g. `data`, `parsed`, etc) to hold the converted value. `Attr` would define it as an accessor over `this.value`, but authors can override it so that it does different things.

This approach allows authors to decide for themselves what the source of truth would be.
If they'd prefer, they can even define `data` it as a class field, with `value` being the accessor that proxies it.

### API surface

Many native features add methods etc to the element.
E.g. the `popover` attribute also adds `showPopover()`.

However, just like reflection, trying to specify this adds additional complexity, and is not strictly necessary:
with the model of `Attr` subclasses, authors can always hang methods on their `Attr` subclass, and they will be accessible via `element.attributes.attrName.methodName()`.

Authors can use additional JS features to improve ergonomics, such as [first-class protocols](https://github.com/tc39/proposal-first-class-protocols), [decorators](https://github.com/tc39/proposal-decorators),
or even monkey-patching, at their own risk.

> [!Important]
> It doesn't look like we need a primitive for this.

If we want to make things easier, we *could* have a lifecycle hook for registration that lets authors react to the attribute being registered on an element.

```js
class MyAttr extends Attr {
	// elided

	static definedCallback(name, ElementConstructor) {
		console.log(name, ElementConstructor.name);
	}
}

HTMLInputElement.customAttributes.define("my-attr", MyAttr);
// prints my-attr HTMLInputElement
```

### Scoping

Some proposals involve a global `customAttributes` registry, while in others `customAttributes` is a property of specific element classes, with `HTMLElement` serving as the global one.

However, many (most?) use cases only involve specific element types and don't make sense in the global scope.

Additionally, the **same attribute** name may have entirely different meanings depending on the context (e.g. `for` is often used generically for element linking, and can mean completely different things).

And of course, the larger the scope, the larger the potential for clashes.

Global attributes would need to also work on SVG, MathML etc, which could delay the entire feature if global attributes are the MVP we go with.

> [!Important]
> Given the number of use cases around specific element types and the complexity of handling SVG at this early stage, it seems prudent to **scope to specific element classes**.

Additionally, as @annevk [points out](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3597830700):
> `CustomElementRegistry` can be scoped to documents, shadow roots, and **elements**. And `document.customElementRegistry` is probably what we want to mimic for anything new. Not sure we should add another global accessor for this.

@sorvell also [talked](https://github.com/WICG/webcomponents/issues/1029#issuecomment-1718332785) about scoped registries:
> Experience with customElements and the scoped registries proposal suggests that scoping is a must and to avoid the pain custom elements has gone through, this feature shouldn't ship without it.
While it's clear that

However, given the amount of time it took to ship scoped registries for custom elements, it does not seem prudent for this to be a blocker.
Nothing prevents us from shipping scoped custom attributes later.

> [!Important]
> Let’s defer scoped custom attributes for later.

### Same attribute on multiple element types

There are many use cases where **the same attribute needs to apply to multiple element types**, without it being global.
Examples abound in the platform: `href`, `src`, several form control attributes, loading attributes like `loading` or `crossorigin`, etc.

Therefore, it should be possible to **register the same attribute to multiple classes**, since it is not always feasible to use inheritance to register an attribute on multiple elements.

E.g. consider a `persist-value` attribute that is placed on form controls to persist their values in localStorage whenever they are edited.
We may want to register it on built-ins like `HTMLInputElement`, `HTMLTextAreaElement`, `HTMLSelectElement` by default:

```js
// persist-value.js
export class PersistAttr extends Attr {
	// elided
}

// Add to native form elements by default
HTMLInputElement.customAttributes.define("persist-value", PersistAttr);
HTMLTextAreaElement.customAttributes.define("persist-value", PersistAttr);
HTMLSelectElement.customAttributes.define("persist-value", PersistAttr);
```

Then, consumers may want to additionally register it on custom form controls they use:
```js
import { PersistAttr, RangeSlider } from "./attrs/persist-value.js";

RangeSlider.customAttributes.define("persist-value", PersistAttr);
```

### Naming

Originally, **hyphens** were suggested, as a way to mirror custom element naming rules.
However, there are many exceptions in the web platform making this a bit awkward, e.g.:

- A ton of SVG presentation attributes (e.g. `fill-opacity`)
- `aria-*`
- `data-*`
- `allow-charset`
- `http-equiv`

If we scope down v1 to `HTMLElement` only, we are left with a fixed, manageable set of names to exclude: Anything starting with `aria-`, as well as the two existing attributes.
`data-*` does not need to be excluded, since it's already authorland.

Another issue making this hard is that many custom element attributes use hyphens.
On two different social media polls, about half of authors voted that they prioritize readability over [platform consistency](https://www.w3.org/TR/design-principles/#naming-consistency) (which recommends concatcase):
- https://x.com/LeaVerou/status/1863812496106389819
- https://front-end.social/@leaverou/113587139842885936

<!--
Additionally, as @jakearchibald [pointed out](https://github.com/WICG/webcomponents/issues/1029#issuecomment-2454991200), even attributes without dashes often camelCase in JS, which would make reflection clash (e.g. `readonly` → `readOnly`).
Of course that is not a problem if we don't handle reflection automatically.
-->

Another option would be a specific **prefix**, though given that the attribute name itself often needs to be namespaced with a library prefix, this would only produce acceptable ergonomics if very short, e.g. a single character (`#`, `$`, `:`, `@` etc).

The CSS custom ident prefix (`--`) has also been proposed.
On one hand it is the target of numerous author complaints, on the other it _is_ an existing established convention.


### Lifecycle hooks

What does `connectedCallback` etc mean in the context of an attribute?
Do they still correspond to the *element* being connected, or the *attribute* being specified on the element? Or when both are true?

@DeepDoge [makes a good case](https://github.com/WICG/webcomponents/issues/1029#issuecomment-3599188181) for the latter:

> IMO custom attributes are composable behavior units, kind of like a superset of extended custom elements. So, `connectedCallback()` should run only when both are true:
>
> * The attribute is attached to an element
> * That element is connected to the DOM
>
> If we simplify it even more, it should trigger when the attribute is connected to the DOM, not when attribute is connected to an element.
>
> Just like how a custom element's `connectedCallback()` gets triggered when the element is actually connected to the DOM, not when it has a parent.
>
> The whole point of `connectedCallback()` / `disconnectedCallback()` is to initialize or clean things up depending on whether the thing (element or attribute) is live in the DOM.
>
> So should it be called "when the attribute is connected, or when the element is connected?": It should be called when the attribute is connected to the DOM, which also requires element its connect to be connected to the DOM as well. I think we can all agree that "Connected" means "Connected to the DOM".

We could probably define similar semantics for other lifecycle callbacks (`disconnectedCallback`, `adoptedCallback` etc), though they may be less straightforward.

It would be good to do a review of use cases to see how common it is to need lifecycle hooks around the element itself, that are separate from those of the attribute node. Assuming these are niche, they can always be addressed via `MutationObserver` improvements down the line (e.g. [observing connectedness is already an open feature request](https://github.com/whatwg/dom/issues/533))

> [!Important]
> Let's define lifecycle callbacks taking into account the element and the attribute as a whole.
> Review use cases to see if element-specific hooks are needed.


### How to react to attribute changes?

While it may seem at first that simply specifying a setter on `attr.value` would help us react to attribute changes, that is not what actually happens today ([demo](https://codepen.io/leaverou/pen/azNQJdP?editors=1012)).
The getter of `attr.value` simply returns the value of an internal slot, so setting attribute values does not go through it at all.

While a mutation observer is always an option, reusing `attributeChangedCallback()` seems like a very fitting solution, and on par with reusing existing lifecycle hooks.

> [!Important]
> `attributeChangedCallback()` will fire when the attribute changes.

### Traits involving multiple attributes

While not MVP, there are many use cases where a feature utilizes multiple attributes.
A common pattern is when one attribute enables the feature, and the rest customize it.

For example, in the web platform there is `<template shadowrootmode="open">`, but also several `shadowroot*` attributes that provide parameters (`shadowrootdelegatesfocus` etc).

Another pattern is where multiple attributes work together to specify a DSL. For example [Vue directives](https://vuejs.org/api/built-in-directives.html) (`v-if`, `v-for`, `v-on` etc).

Therefore, a nice-to-have would be to have a way for the attribute to react to attribute changes of *other* attributes on the element.
And if we're already using `attributeChangedCallback()` (see above),
we may as well reuse `observedAttributes` and feed two birds with one scone (note that the attribute _itself_ would always be observed automatically, authors would only need `observedAttributes` for observing *other* attributes).

> [!Important]
> Let's reuse `attributeChangedCallback()` and `observedAttributes()` to allow an attribute to observe others.

## Current Proposal

Putting all of the above together gives us a strawman to facilitate discussion.

### Summary

Custom attributes are defined as a subclass of `Attr` and registered for use with one or more element constructors:

```js
class MyTooltip extends Attr {
	// elided
}

HTMLElement.customAttributes.define("my-tooltip", MyTooltip);
```

### New members on existing interfaces

#### `HTMLElement.customAttributes`

New static member of type `CustomAttributeRegistry`, with the same methods as `CustomElementRegistry`. This is a distinct instance per subclass, and an element recognizes attributes registered on any of its superclasses.

> [!Note]
> We may want to define a `CustomRegistry` superclass that they both inherit from.

### `Attr` subclass members

### Lifecycle hooks

Lifecycle callbacks similar to custom elements are available:

- `connectedCallback`: Executed when the attribute is present on the element and `this.ownerElement.isConnected` is true.
- `disconnectedCallback`: Executed when the attribute is no longer connected (see above)
- `connectedMoveCallback()`: TBD
- `adoptedCallback()`: Fired when the attribute node is moved to another element (e.g. via `setAttributeNode`) OR `ownerElement` is moved to another document.
- `attributeChangedCallback()`: Executed when the attribute itself or any of the attributes in `this.constructor.observedAttributes` are changed, added, removed, or replaced. The attribute itself is always observed whether it’s specified in `observedAttributes` or not.

There is also a static lifecycle hook:
- `definedCallback(name, ElementConstructor)`: Executed whenever the attribute is defined on an element constructor

## Notes / Patterns

### Persistent attribute node

Note that browsers currently create a new `Attr` node every time an attribute is added, even though they reuse an existing node if an existing attribute value is changed.

If reusing the same node is desirable (e.g. due to high setup costs), this could be done with a `WeakMap`:

```js
/** @type WeakMap<HTMLElement, Attr> */
let nodes = new WeakMap();

class MyAttr extends Attr {
	constructor() {
		super();

		let existing = nodes.get(this.ownerElement);

		if (existing) {
			return existing;
		}
	}
}
```

### Timing

Currently, `Attr` nodes are already constructed by the time the constructor of a custom element’s subclass runs,
presumably by `Element`’s constructor ([demo](https://codepen.io/leaverou/pen/azNQWoz?editors=1012)).

Should upgrading happen then too?
This means any custom attribute code needs to run before the element has a chance to construct itself fully.

We could also define it to run after the constructor it was registered on.
E.g. registering an attribute on `HTMLElement` would run it after the `HTMLElement` constructor, whereas registering an attribute on `HTMLFormElement` would run it after the `HTMLFormElement` constructor.

## FAQ

### Does this replace custom elements?

No, they have distinct purposes, just like attributes and elements have distinct purposes in the platform.
Some things are better suited to attributes, and others to elements.

For example, you wouldn't want to implement a text field by doing `<div my-textfield textfield-value="foo" textfield-autofocus></div>`.
Ew!
An element is or isn't a text field, it's not something you can just slap on any element.

That said, *specializing* an element type is a totally valid use case. E.g. `<input type="password" pwd-toggle>` or even `<button my-button>`.
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
