- Start Date: 2017-02-07
- RFC PR: https://github.com/elix/rfcs/pull/7
- Elix code PR: https://github.com/elix/elix/pull/5


# Summary

This RFC proposes two mixins and a package of helper functions for components
that want to programmatically inspect or manipulate their own contents:

* **DefaultSlotContentMixin**.
  Defines a component's content as the nodes assigned its default (unnamed)
  slot.
* **ContentItemsMixin**.
  Lets a component treat its content as items in a list.
* **content** helper functions.
  Help retrieve a component's children in a variety of ways that meet Gold
  Standard checklist criteria.


# Motivation

The following Gold Standard checklist requirements pertain to the way components
interact with their light DOM contents:

* [Child Independence](https://github.com/webcomponents/gold-standard/wiki/Child-Independence):
  Can you use the component with a wide range of child element types?

* [Content Assignment](https://github.com/webcomponents/gold-standard/wiki/Content-Assignment):
  Can you place a <slot> element inside a component instance and have the
  component treat the assigned content as if it were directly inside the
  component?

* [Content Changes](https://github.com/webcomponents/gold-standard/wiki/Content-Changes):
  Will the component respond to runtime changes in its content (including
  distributed content)?

* [Auxiliary Content](https://github.com/webcomponents/gold-standard/wiki/Auxiliary-Content):
  Does the component permit the use of child elements that perform auxiliary
  functions?

Since many components accept contents, mixins can provide a standard and
efficient means to satisfy the above criteria.

Desired outcomes:

* All Elix components that work with content support the above criteria.
* Developers can easily support the above criteria in their own components.


# Use cases

`DefaultSlotContentMixin` is aimed at components intended to contain light DOM
children. For a general-purpose library like Elix, that includes most of the
project's envisioned components.

`ContentItemsMixin` is aimed at the same set of list-like elements as
`SingleSelectionMixin`. The set of list items will usually — but not always —
be extracted from the component's light DOM children.


# Detailed design

## DefaultSlotContentMixin

This mixin defines a property called `symbols.content` that returns all
nodes assigned to the component's default (unnamed) slot. This implies the
component has created a shadow subtree which includes such a slot; a console
warning is emitted if no such slot is found.

This also provides notification of changes to the component's `symbols.content`
property. It will invoke a `symbols.contentChanged` method when the component is
first instantiated, and whenever the elements assigned to its default slot
change. This is intended to satisfy the Gold Standard checklist item for
monitoring [Content
Changes](https://github.com/webcomponents/gold-standard/wiki/Content-Changes).

Example:

```
class CountingElement extends DefaultSlotContentMixin(HTMLElement) {

  constructor() {
    super();
    let root = this.attachShadow({ mode: 'open' });
    root.innerHTML = `<slot></slot>`;
    this[symbols.shadowCreated]();
  }

  [symbols.contentChanged]() {
    if (super[symbols.contentChanged]) { super[symbols.contentChanged](); }
    // Count the component's children, both initially and when changed.
    this.count = this.assignedChildren.length;
  }

}
```

To receive `contentChanged` notification, this mixin expects a component to
invoke a method called `symbols.shadowCreated` after the component's shadow
root has been created and populated. This allows the mixin to inspect the
shadow subtree for `slot` elements, and listen to their `slotchange` events.


## ContentItemsMixin

Mixin which maps content semantics to list item semantics. Items differ from
element contents in several ways:

* They are often referenced via index.
* They may have a selection state.
* It's common to do work to initialize the appearance or state of a new item.
* Text nodes (which are often whitespace) are ignored.
* Invisible child elements are filtered out and not counted as items.
  Instances of invisible `Node` subclasses such as `Comment` and
  `ProcessingInstruction` are filtered out, as are invisible auxiliary elements
  include link, script, style, and template elements. This filtering ensures
  that those auxiliary elements can be used in markup inside of a list without
  being treated as list items.

This mixin expects a component to provide a `symbols.content` property returning
a raw set of elements. This can be done with `DefaultSlotContentMixin` (above),
or defined by hand. A component could, for example, define `symbols.content` to
only extract nodes assigned to a particular `slot`. For a list-like component
whose list items are hard-coded, the component could define `symbols.content` to
return the children of an element inside the component's shadow subtree.

This mixin defines a property called `items` that filters the results of
`symbols.content`. This is an array of items designed for use with
`SingleSelectionMixin` and its companion mixins. The mixin constructs a value
for the `items` property that is equal to `symbols.content` — minus anything
that would not normally be visible to the user. Example:

    <my-element>
      <style>
        div {
          color: gray;
        }
      </style>
      <div>One</div>
      <div>Two</div>
      <div>Three</div>
    </my-element>

If this element uses `ContentItemsMixin`, its `items` property will return the
three `div` elements, and filter out the text nodes and the invisible `style`
element. This filtering is done through the helper function
`content.substantiveElements` (below).

To allow a component to initialize items, the mixin implements a handler for
`symbols.itemsChanged` that invokes `symbols.itemAdded` for any new items added
since the last `itemsChanged` call. The component can then perform per-item
initialization work in `itemAdded`.

To avoid having to recalculate the set of `items` each time that property is
request, this mixin supports an optimized mode. If the method
`symbols.contentChanged` is invoked, the mixin concludes that the component will
notify it of future content changes, and turns on the optimization. In that
mode, the mixin saves a reference to the computed set of items, and will return
that immediately on subsequent calls to the `items` property. If this mixin is
used in conjunction with `DefaultSlotContentMixin` (above), the latter will take
care of automatically invoking `symbols.contentChanged`, and automatically
engage the optimization.


## content helpers

This is a collection of helper functions for accessing a component's content.
For now, this provides the following helper function:

* `substantiveElements`: given a `NodeList` or `Array` of Nodes, this
  returns only those Elements that can typically be seen by a user.

The above filters out:

1. Text nodes
2. Invisible `Node` subclasses like `Comment` and `ProcessingInstruction`.
3. Elements that can appear in the document body, but which typically have no
   user-visible manifestation.

The third criteria is achieved through use of a blacklist. Consulting the list
of [all HTML elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element),
the blacklist of invisible auxiliary elements appears to be:

* `embed`
* `link`
* `noscript`
* `object`
* `param`
* `script`
* `source`
* `style`
* `template`

For completeness, the blacklist also filters out these deprecated auxiliary
elements:

* `applet`
* `basefont`
* `font`
* `frame`
* `frameset`
* `isindex`
* `keygen`
* `multicol`
* `nextid`
* `noembed`


# Alternatives

As with most Elix mixins, we could decide to have each component define its own
means of retrieving content and filtering out the useful items. However, the
needs here appear to be so consistent that it would be cumbersome to manage this
on a per-component basis.
