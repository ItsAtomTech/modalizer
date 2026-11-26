# Modalizer JS

**Version:** 1.4  
**Author:** Atomtech  
**Files:** `modalizer.js` · `modalizer.css`

**Documentation Co-Authored with Claude**
---

## Overview

Modalizer is a lightweight JavaScript library that brings a **modularized page-render approach** to non-SPA backends like Flask, Django, or any traditional server-rendered stack.

Instead of navigating between full pages, Modalizer loads sub-pages inside tracked iframes — giving you the feel of a component-based frontend without rebuilding your backend. It manages the full iframe lifecycle: creation, tracking, handshake verification, cross-frame messaging, minimize/restore, and cleanup.

---

## How It Works

Each "modal" is an `<iframe>` injected into the DOM and wrapped in a container div. When a page is opened as a modal, Modalizer:

1. Creates and injects the iframe into the DOM (or reuses an existing one by ID)
2. Begins a handshake timer — the loaded page must respond with `"OK"` via `postMessage` within the timeout window
3. If the handshake succeeds, the modal is tracked as active
4. If it fails, Modalizer auto-closes the modal and shows a failure toast

This makes it reliable for backend routes that may be slow or fail silently.

---

## Installation

Include both files in your base HTML template:

```html
<link rel="stylesheet" href="modalizer.css">
<script src="modalizer.js"></script>
```

> Modalizer must run in the **top-level window** — it will refuse to initialize inside an iframe.

---

## Quick Start

### Opening a Modal

Call `open_modal()` with the URL of any backend route:

```html
<button onclick="open_modal('/dashboard/users')">Open Users</button>
```

The target route renders as a full-page iframe overlaid on the current page.

### Closing from Inside the Modal

Inside the loaded iframe page, call:

```javascript
postMessageToParent('close');
```

Or use the built-in close button that Modalizer injects automatically into every modal.

### Handshake (Required in Child Pages)

Every page loaded as a modal **must** include `modalizer.js`. On load, it automatically sends the handshake:

```javascript
postMessageToParent("OK"); // sent automatically by modalizer.js
```

Without this, Modalizer will auto-close the modal after the timeout and show `"Failed to Load Page..."`.

---

## API Reference

### `open_modal(link, custom_class, targetElm, dialog, onclose, prevent_reload)`

Opens a URL as a tracked iframe modal.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `link` | `string` | — | URL of the page to load |
| `custom_class` | `string` | `undefined` | Comma-separated CSS class(es) to apply to the modal container |
| `targetElm` | `Element` | `undefined` | DOM element to append the modal into. Defaults to `document.body` |
| `dialog` | `boolean` | `false` | Uses a native `<dialog>` element instead of a `<div>` |
| `onclose` | `function` | `undefined` | Callback fired when the modal is removed from the DOM |
| `prevent_reload` | `boolean` | `false` | If `true` and the modal is already open, sends a `refreshEvent` instead of reloading the iframe |

**Returns:** The modal container element.

**Example:**

```javascript
open_modal(
  '/reports/monthly',
  'floating_modal,smaller_modal',
  document.getElementById('workspace'),
  false,
  () => console.log('Modal closed'),
  true
);
```

---

### `close_modalizer(elm)`

Closes a modal given a reference to its close button element (or any child element inside the modal container).

```javascript
close_modalizer(this); // used in onclick handlers
```

Plays the `zoomOutWithCircle` animation before removing the element from the DOM.

---

### `minimize_modalizer(elm)`

Minimizes (hides) a modal without destroying it. The iframe stays alive in the DOM with `display: none`.

```javascript
minimize_modalizer(this);
```

Calling `open_modal()` with the same URL again will restore it and play the `slideIn` animation.

---

### `postMessageToParent(message)`

Sends a message string from a child iframe to the parent window.

```javascript
postMessageToParent('close');           // close this modal
postMessageToParent('goToParent:/home'); // navigate the parent window
```

---

### `postMessageToModal(id, data)`

Sends a message string from the parent to a specific open modal by its ID.

```javascript
postMessageToModal('md_dashboardusers', 'refreshEvent');
```

---

## Cross-Frame Messaging

Modalizer listens for `postMessage` events from child iframes and handles the following built-in commands:

| Message | Action |
|---|---|
| `"close"` | Closes the modal the message originated from |
| `"OK"` | Confirms handshake — stops the auto-close timer |
| `"goToParent:<url>"` | Navigates the parent window to the given URL |
| `"refreshEvent"` | Triggers `REFRESH_HANDLER` if defined |
| `"minimized"` | Sets internal `isMinimized` flag |
| `"openModal:<json>"` | Opens a new modal from inside a child iframe |
| `"function:<name>:<params>"` | Calls a named function on the parent window |

### Opening a Modal from Inside a Modal

A child iframe can instruct the parent to open another modal:

```javascript
postMessageToParent(
  'openModal:' + JSON.stringify({
    link: '/users/edit',
    custom_class: 'floating_modal',
    target: 'workspace',
    dialog: false
  })
);
```

### Refresh Handler

Define `REFRESH_HANDLER` on the parent window to respond to refresh events from child iframes:

```javascript
REFRESH_HANDLER = function() {
  reloadTable(); // your custom logic
};
```

---

## CSS Classes Reference

These are the classes Modalizer applies programmatically. Apply them via `custom_class` or your own logic to control modal appearance.

| Class | Applied by | Description |
|---|---|---|
| `.modal_iframe_con` | JS (always) | Root wrapper for every modal |
| `.modal_iframe` | JS (always) | The `<iframe>` element itself |
| `.loading_frame` | JS (always) | "Loading" overlay shown before handshake |
| `.close_modal_rev` | JS (always) | The injected close button |
| `.native_dialoge` | JS (when `dialog=true`) | Applied when using a `<dialog>` element |
| `.fade_out_circle` | JS on close/minimize | Triggers the zoom-out circle animation |
| `.fade_in_top` | JS on restore | Triggers the slide-in animation on restore |
| `.minimized_modal` | JS on minimize | Hides the modal container (`display: none`) |
| `.blurred` | Manual / custom_class | Applies `backdrop-filter: blur(2px)` to the modal |
| `.no_close_button` | Manual / custom_class | Hides the injected close button |
| `.full_modal` | Manual / custom_class | Moves the close button to top-left (full-screen layout) |

---

## Flask Integration Example

A typical Flask setup with Modalizer — the base layout holds the Modalizer scripts, and each route renders a standalone page that can be loaded as a modal.

**`base.html`**
```html
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="{{ url_for('static', filename='modalizer.css') }}">
</head>
<body>
  <button onclick="open_modal('/users')">Open Users</button>

  <script src="{{ url_for('static', filename='modalizer.js') }}"></script>
</body>
</html>
```

**`users.html`** *(the page loaded inside the modal)*
```html
<!DOCTYPE html>
<html>
<body>
  <h2>Users</h2>
  <!-- page content -->

  <button onclick="postMessageToParent('close')">Close</button>

  <!-- Required: must include modalizer.js for the handshake -->
  <script src="{{ url_for('static', filename='modalizer.js') }}"></script>
</body>
</html>
```

**`app.py`**
```python
@app.route('/users')
def users():
    return render_template('users.html')
```

---

## URL-Based Deep Linking

Modalizer supports auto-opening a modal via the `?m=` query parameter. Add `?m=pagename` to any URL and Modalizer will try to click the element with `id="_pagename"` on load.

```html
<!-- This element will be auto-clicked when ?m=users is in the URL -->
<button id="_users" onclick="open_modal('/users')">Users</button>
```

Navigating to `https://yourapp.com/?m=users` will automatically open the Users modal.

---

## Loading Spy Implementation

If you want to use the built-in loading spy feature, it is required that you put the following html DOM element somewhere on your base or parent shell page:

```html
<div class="" id="load_spy">
	<div class=""> </div>
</div>

```
### Activating on parent window 

```js
ALLOW_PROGRESS_SPY = true;
```

Also, you need to set this variable on the parent page as well, which is FALSE by default to prevent child views from trying to display its own loader spy element.
You must declare it after the script lib gets loaded or it would not apply at all or even cause reference error. 

---

## Keyboard Behavior

Modalizer overrides the default browser refresh shortcuts (`F5`, `Ctrl+R`) with context-aware behavior:

- **Inside a modal (iframe):** refreshes only the iframe content (`window.location.href = ""`)
- **On the parent page:** reloads the full page as normal (`window.location.reload()`)

---

## Lifecycle Diagram

```
open_modal(url)
      │
      ├─ Modal already open? ──► reuse / reload / refreshEvent
      │
      └─ Create container + iframe
              │
              ▼
        Inject into DOM
              │
              ▼
        Start handshake timer (10 ticks × 1s)
              │
         ┌────┴────┐
         │         │
      "OK" msg   Timer hits 0
      received      │
         │        auto-close + showToast("Failed to Load Page...")
         ▼
      Modal active & tracked
              │
    ┌─────────┼──────────┐
    │         │          │
  close    minimize   refreshEvent
    │         │
  fade out  hide (display:none)
  + remove  stays in DOM
              │
           restore via open_modal() again
              │
           fade_in_top animation
```

---

## Notes

- Modalizer is designed for the **parent window only**. All `open_modal` / `close_modalizer` / `minimize_modalizer` calls are no-ops (with a console warning) when called from inside an iframe.
- Modal IDs are derived from the URL with all non-alphanumeric characters stripped: `/users/edit` becomes `md_usersedit`.
- The `?m=` deep-link feature depends on a `getparam()` helper and an `_()` selector helper being available globally. Define them in your base template if not already present:

```javascript
function getparam(name) {
  return new URLSearchParams(window.location.search).get(name);
}
function _(id) {
  return document.getElementById(id);
}
```
