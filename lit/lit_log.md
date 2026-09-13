# Lit Learning Log

A running personal collection of Lit rules of thumb, good practices, and lessons learned, gathered day after day while working on real projects (but written generically so they're useful anywhere).

Entries are logged chronologically (oldest first). This log will be reorganized into logical chapters once it grows large enough to need them.

## Index

- [1. Public vs private vs constructor-only properties](#1-public-vs-private-vs-constructor-only-properties)
- [2. Property reflection (reflect: true)](#2-property-reflection-reflect-true)

## Entries

### 1. Public vs private vs constructor-only properties

Lit gives you three levels of property declaration, each with different visibility and reactivity characteristics. Choosing the right one avoids unnecessary re-renders and keeps your component's API clean.

**Public reactive properties** (`@property()` or `{ type: String }` in `static properties`) are the component's public API. They can be set via HTML attributes or JavaScript, and any change triggers a re-render. Use them for data that consumers of your component are expected to pass in.

**Private reactive properties** (`@state()` or `{ state: true }`) are internal to the component but still trigger a re-render when they change. Use `state: true` when the property holds internal state that the template depends on — for example, a loading flag, a toggle state, or fetched data that the template renders. Because `state: true` suppresses attribute handling, these properties cannot be set from the outside via HTML.

**Constructor-only variables** (plain class fields or assignments in `constructor()`) are not declared in `static properties` at all. They do **not** trigger re-renders when mutated. Use them for bookkeeping data that the template never reads — timeout IDs, cached DOM references, AbortControllers, internal counters for logic, or configuration that never changes after initialization.

| Scenario | Declaration | Triggers render? | Settable from outside? |
|----------|-------------|-----------------|----------------------|
| Consumer passes data in | `@property()` | Yes | Yes (attribute + JS) |
| Internal state shown in template | `@state()` / `state: true` | Yes | No |
| Internal bookkeeping, not rendered | Constructor variable | No | No |

**Should every private property use `state: true`?** No. Only use it when changing that value must cause the component to update its DOM. If the variable is just housekeeping (a timer ID, a reference to a child element, a request controller), declaring it reactive would cause pointless re-renders every time you reassign it.

```js
import { LitElement, html } from 'lit';
import { property, state } from 'lit/decorators.js';

class UserCard extends LitElement {
  // Public: consumers set this via <user-card .userId=${id}>
  @property({ type: String }) userId = '';

  // Private reactive: internal, but template reads it → needs re-render
  @state() _userData = null;
  @state() _loading = false;

  // Constructor-only: bookkeeping, template never shows this
  _abortController = null;

  async connectedCallback() {
    super.connectedCallback();
    this._abortController = new AbortController();
    this._loading = true; // triggers render → shows spinner
    this._userData = await fetchUser(this.userId, this._abortController.signal);
    this._loading = false; // triggers render → shows data
  }

  disconnectedCallback() {
    super.disconnectedCallback();
    this._abortController?.abort(); // no render needed
  }

  render() {
    if (this._loading) return html`<spinner-el></spinner-el>`;
    return html`<p>${this._userData?.name}</p>`;
  }
}
```

**Further reading:** [Lit documentation: Define reactive properties](https://lit.dev/docs/components/properties/#defining-properties)

### 2. Property reflection (reflect: true)

In Lit, property changes normally trigger a component re-render, but they do not automatically update the corresponding HTML attribute in the DOM. Setting `reflect: true` instructs Lit to automatically serialize and write the property's value back to the element's DOM attribute whenever it changes. This creates a two-way synchronization between the JavaScript property and the HTML attribute.

Reflecting properties is highly useful, but it comes with performance overhead because Lit must convert the value to a string and execute a DOM write. Therefore, reflection should be used selectively.

#### When to Use Reflection
- **CSS Styling based on Host State**: When you need to style the component's host element using attribute selectors like `:host([disabled])` or `:host([state="inactive"])`.
- **Accessibility and Semantics**: When syncing properties with standard ARIA attributes (e.g., `aria-expanded`) or standard semantic attributes (e.g., `disabled`, `open`).
- **External Interoperability**: When other testing frameworks, analytical tools, or non-reactive legacy scripts need to inspect the current state of your element directly from the DOM using `element.getAttribute('state')` or query selectors.

#### When NOT to Use Reflection
- **Objects and Arrays**: Serializing large objects or arrays back to attributes is extremely slow and results in bloated HTML.
- **Internal State**: If a state change only affects internal templates rendered inside `shadowRoot`, keeping it in JS-only memory is much faster.

```js
import { LitElement, html, css } from 'lit';

class AccordionItem extends LitElement {
  static get properties() {
    return {
      // Reflect 'open' to attribute so we can apply styles to the host element
      open: { type: Boolean, reflect: true },
      // Reflect 'title' is unnecessary because it is only used to render content
      title: { type: String },
    };
  }

  static get styles() {
    return css`
      /* style the host element when 'open' attribute is present */
      :host([open]) {
        border-color: blue;
      }
    `;
  }

  constructor() {
    super();
    this.open = false;
    this.title = '';
  }

  toggle() {
    this.open = !this.open;
  }

  render() {
    return html`
      <div class="header" @click=${this.toggle}>${this.title}</div>
      <div class="content">Content...</div>
    `;
  }
}

customElements.define('accordion-item', AccordionItem);
```

**Further reading:** [Lit documentation: Property Reflection](https://lit.dev/docs/components/properties/#reflection)
