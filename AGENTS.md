# Agent guide: Subscribe & Save Upgrade for the Shopify Cart

Instructions for AI coding agents (Claude Code, Cursor, Codex, etc.) installing or customizing this section. Humans should start with [README.md](README.md).

## What this is

A single Shopify theme section, `sections/cart-subscribe-upgrade.liquid`. Inside the theme's **cart drawer** it adds a card under every line whose product has a selling plan:

The theme's row and our strip are outlined as **one card** (layout A): the strip is the card's bottom edge.

- **One-time line:** the strip is one `<button>`: headline ("Subscribe & save 15%"), a line with price, frequency and savings, and a **Subscribe** pill as the visible call to action. One tap switches the line to a subscription with the Cart AJAX API.
- **Subscription line:** the strip shows a frequency `<select>`, the savings per delivery and a "Switch to one-time" link. The theme's own line price gets the one-time price crossed out in front of it.

No app. It works with any subscription app that uses Shopify selling plans (tested with Shopify Subscriptions).

```
sections/cart-subscribe-upgrade.liquid   The product. Markup, CSS, JS and schema in one file.
README.md                                Install steps, settings, troubleshooting
```

The section never creates selling plans or discounts. Plans come from the merchant's subscription app. Nothing in this repo can create them: Shopify CLI has no commands for that, and doing it from storefront JS would need an Admin API token, which must never ship in a theme.

## Installing into a theme

With Shopify CLI, from the user's theme directory:

1. Copy `sections/cart-subscribe-upgrade.liquid` into the theme's `sections/` folder.
2. Add it to the **footer group** (it renders nothing visible there; it mounts into the cart drawer). Pull the file first so you don't overwrite the merchant's footer edits:
   ```bash
   shopify theme pull --only sections/footer-group.json
   ```
   Then add under `sections`, and append the key to `order`:
   ```json
   "cart_subscribe_upgrade": { "type": "cart-subscribe-upgrade", "settings": {} }
   ```
3. Make sure the theme's cart type is **drawer** (Horizon and Dawn: `settings.cart_type == "drawer"` in `config/settings_data.json`).
4. Push to an **unpublished** theme first: `shopify theme push --unpublished --theme "Subscribe upgrade"`, or `--theme <id> --only sections/cart-subscribe-upgrade.liquid --only sections/footer-group.json`.
5. Ask the human before pushing to the live theme (`--allow-live`).
6. Check that at least one product has a selling plan with a discount: `GET /products/<handle>.js` should have a non-empty `selling_plan_groups`. If not, the human sets up a plan in their subscription app.

## Hard rules when changing the section

These keep it portable. Breaking one usually works in the theme you're testing and fails in every other theme.

1. **Stay one file.** CSS goes in `{% stylesheet %}`, JS in `{% javascript %}`. No new files in `assets/` or `snippets/`. Neither block can contain Liquid; all settings reach the JS through the JSON in `<script type="application/json" data-csu-config>`.
2. **No theme dependencies for core behavior.** Theme-specific selectors are allowed only as defaults in `DEFAULTS` or as optional extras that do nothing when they don't match.
3. **Prefix everything.** Classes `csu-*` / `csu__*`, CSS variables `--csu-*`, custom element `<cart-subscribe-upgrade>`, events `csu:*`, request header `X-CSU`, attributes `data-csu-*`.
4. **Every shopper-facing string is a setting.** Schema labels are plain English, not `t:` keys, because the section can't ship locale files.
5. **Guard the element definition**: `if (customElements.get('cart-subscribe-upgrade')) return;`.
6. **Never hardcode the discount or frequencies.** Prices come from Shopify (`selling_plan_allocations` in product JSON, `selling_plan_allocation` in cart JSON), never from our own percent math. The percent label is the only computed value, and only when the plan's adjustment isn't a percentage.
7. **Keep Liquid out of `{% javascript %}`**, including literal `{{ }}` in strings. The money-format fallback is built with string concatenation on purpose.
8. **Everything injected into theme markup carries `data-csu-owner="<section id>"`**, and attributes we set on theme elements (`data-csu-card`, `data-csu-hidden`) hold the section id, so `cleanup()` can remove all of it when the theme editor re-renders the section.

## How it works

### Liquid (top of the file)
Builds one JSON config: money format, which plan the button picks (`planPosition`), the display toggles (`itemCard`, `strikeLines`, `hidePlanName`), all strings, colors and the 4 advanced selectors. `applyColors()` turns the colors into CSS variables in one `<style data-csu-owner>` in `<head>`, scoped to this instance's strips and card rows.

### Data
- `GET /cart.js` for the lines. Each line has `key`, `quantity`, `variant_id`, `handle`, `final_line_price` and `selling_plan_allocation` (null when one-time; for subscriptions it has `price`, `compare_at_price` and `selling_plan`).
- Cart JSON doesn't include a product's plans, so `product(handle)` fetches `/products/<handle>.js` once per handle per page load (cached promise). It reads `selling_plan_groups[].selling_plans[]` (id, `options[0].value` for the label, `price_adjustments`) and `variants[].selling_plan_allocations[]` (`price`, `compare_at_price`, `per_delivery_price` per plan).
- `model(item)` turns one line into what the card shows, or `null` for no card:
  - No card for $0 lines (free gifts), nested/bundle lines, products with no plan for that variant, and one-time lines where the default plan saves nothing.
  - Plans come from the group that holds the line's current plan, else the first group with an allocation for the variant. Prepaid plans (`price != per_delivery_price`) are skipped because "save on every delivery" wouldn't be true.
  - Default plan: position from `planPosition` (first / second / third / last), clamped to the last plan.
  - Frequency label: the plan's first option value with a leading "Deliver every" / "Delivery every" / "Every" removed ("Deliver every 4 weeks" becomes "4 weeks"), else the plan name.
  - Savings = (one-time unit price minus plan unit price) × quantity.

### Mounting
`<cart-subscribe-upgrade>` stays hidden in the footer. Cards are kept in a `Map` by line key and re-attached, not rebuilt, so redraws don't flicker or lose listeners.
- Drawer: first match of the **drawer selector**.
- Row: `[data-key="<line key>"]` inside the drawer (Horizon), else the nth row (by **row selector**) holding a link to `/products/<handle>`.
- In a `<tr>` the strip goes in a new `<td class="csu-cell">`; elsewhere the strip itself is appended. `.csu-cell` has `grid-column: 1 / -1`, so in Horizon's grid row it spans the full width under the line.
- **The card:** when the row's computed `display` is grid, flex or block (and the **Outline each item** setting is on), the row gets `data-csu-card="<section id>"`. CSS gives it a border, radius and padding (`!important`, since themes style rows with more specific selectors), and the cell bleeds to the card edges with negative margins. A real `display: table-row` can't take a border or radius, so there the strip keeps its own outline instead. The observer also watches `data-csu-card`, so a redraw that resets the row's attributes gets the outline back.
- A `MutationObserver` on `document.body` (ignoring changes inside `[data-csu-owner]`) re-attaches on the next animation frame. Horizon's morph drops unknown nodes on every cart redraw, so this matters. `attach()` is idempotent; when nothing changes it makes no DOM changes, so it settles.

### Knowing when the cart changed
Three sources, all funneled into a debounced `refresh()` that fetches `/cart.js`:
- `shopify:cart:lines-update`, Shopify's standard cart event. Horizon and newer themes dispatch it. Our own events (`detail.source === 'cart-subscribe-upgrade'`) are ignored.
- A one-time `window.fetch` wrapper (`window.__csuFetchPatched`) that fires `csu:cart-changed` after any request to `/cart/add|change|update|clear`. Our requests send `X-CSU` and are skipped. It coexists with the free shipping bar's wrapper (`X-FSB`).
- `cart:updated`, a name some themes use.

### Switching a line (`change`)
- `POST /cart/change.js` with `{ id: <line key>, quantity, selling_plan: <plan id> }` to subscribe or change frequency, and `selling_plan: null` to go back to one-time. Verified on Shopify: the line keeps its quantity and position; its key changes. Shopify merges it into an identical line if one exists.
- The request asks for `sections` = every `cart-items-component[data-section-id]` so the theme can redraw from the response.
- Then `notifyTheme()` dispatches Shopify's `CartLinesUpdateEvent` (imported from `https://cdn.shopify.com/storefront/standard-events.js`, the same module Horizon uses) with the sections HTML, plus a generic `cart:refresh` event, and `load()` rebuilds the cards from the response.
- Busy and error states toggle classes and `disabled`; they never replace the card's HTML mid-click (replacing it detaches the click target, which can make drawers think the click was outside).
- Keyboard focus: if the card had focus, the new card's first control takes it after the redraw, unless focus already moved to another cart line. Horizon parks lost focus on the drawer's close button, which counts as lost.

### Decorating the theme's own markup (`decorate`)
Best effort, re-applied after every redraw:
- **Crossed-out price:** on subscription lines, an `<s class="csu-was">` with the one-time line price is prepended to the first **line price selector** match in the row, and the one-time unit price to the **unit price selector** match (skipped if the theme already has its own `<s>` there). Removed when the line goes back to one-time. If no line price element is found, the subscribed card shows the comparison itself (`.csu__cmp`).
- **Cart total (`decorateTotal`):** Shopify applies a selling plan's discount as a lower line price, not a discount application, so `total_discount` stays 0 and themes show no savings. When any line is subscribed (and **Show subscription savings at the cart total** is on), the first **Cart total value selector** match (tried in order) gets an `<s class="csu-was csu-was--total">` with `total_price` plus the summed subscription savings, and a `<p class="csu-total-saving">` goes right after the total's row. No total found: the line goes after the cart lines' table or list. If the free shipping bar already crossed out the total (`.fsb-was`), ours is skipped and only the savings line is added.
- **Duplicate plan name:** inside a subscribed row that has our card, any leaf element whose text equals the plan name gets `data-csu-hidden` (CSS hides it). Setting: **Hide the theme's plan name**.

### Money
`money()` formats cents with the shop's `money_format` (all Shopify `{{ amount* }}` variants). Product and cart JSON are already in the shopper's presentment currency.

## Supporting another theme

Tested on **Horizon**. Defaults in `DEFAULTS` also cover common Dawn-family names. When something doesn't line up, ask for a screenshot of the open cart drawer, then:

| Symptom | Where to fix |
|---|---|
| No card in the drawer | Find the drawer's root element: **Cart drawer selector** / `DEFAULTS.drawer`. Then check rows: if they have no `data-key`, the fallback needs **Cart line row selector** / `DEFAULTS.row` and a product link inside each row |
| Strip in the wrong place inside the row | Adjust `.csu-cell` CSS for that theme's row layout (grid vs table vs flex), guarded so other themes are unaffected |
| Card outline looks wrong (double borders, clipped corners, odd padding) | The theme styles its rows heavily. Adjust the `[data-csu-card]` rules, or tell the merchant to turn off **Outline each item and its offer as one card** |
| No crossed-out price | **Line price selector** / `DEFAULTS.linePrice`, **Unit price selector** / `DEFAULTS.unitPrice` |
| Line switches but drawer doesn't update until reopened | The theme doesn't listen for `shopify:cart:lines-update`. Add the theme's own refresh call at the end of `notifyTheme()`, guarded so it does nothing in other themes. For Dawn, its `cart-drawer` element has a `renderContents()` method that takes a cart response with `sections` for `cart-drawer` and `cart-icon-bubble`; request those section IDs in `change()` and pass the response to it |
| No savings at the cart total | **Cart total value selector** / `DEFAULTS.total` (most specific selector first) |
| Theme still shows the plan name twice | The theme's text doesn't exactly equal the plan name. Leave it, or hide that element with a theme-specific selector |

Settings are the first choice for a single store. Edit `DEFAULTS` when you're improving the section for everyone.

## Checking your work

There are no automated tests. Verify on a real storefront (preview of an unpublished theme). A password-protected store's preview opens past the password page from the theme editor's **... > Preview**.

1. Add a product with a plan as one-time, open the drawer: the item and strip are one outlined card, with the right percent, frequency, prices and savings (compare with `/products/<handle>.js`).
2. Tap the strip: the line switches without a reload, the theme's price shows the one-time price crossed out, the strip shows the subscribed state.
3. Change the frequency: `/cart.js` shows the new plan.
4. Switch to one-time: full price, crossed-out price gone, button back.
5. Quantity 3: savings and the crossed-out line price triple.
6. A product with no plan: no card.
7. Checkout shows the plan and a recurring subtotal.
8. Phone width: nothing overflows.
9. In the theme editor, change a setting: no duplicate cards or crossed-out prices.
10. Keyboard: focus the strip, press Enter, focus lands on the frequency dropdown. With a mouse, focus is left alone (no focus ring jumps onto the dropdown).
