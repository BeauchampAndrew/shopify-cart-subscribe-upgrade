# Subscribe & Save Upgrade for the Shopify Cart

A big "Subscribe & save" button under every one-time item in your cart drawer. One tap switches that item to a subscription (same product, same quantity) without the shopper leaving the cart. It reads the discount and the delivery frequencies from your products' selling plans, so it works with Shopify Subscriptions or any subscription app that uses them.

It's one file. No extra app, no monthly fee, no code needed to install it.

**Using Claude Code, Cursor or another AI coding tool?** Point it at this repo and say what you want, like "install this in my theme" or "make the button default to every 8 weeks". [AGENTS.md](AGENTS.md) tells it how the section works and how to change it without breaking it in other themes.

## What it does

The shopper with a one-time case in their cart is the easiest one to turn into a subscriber. Most stores only offer Subscribe & Save on the product page, so once the item is in the cart, the offer is gone. This puts it back where they're already looking.

- **The offer is part of the item.** Each item that can subscribe becomes one outlined card, with a green strip along its bottom: "Subscribe & save 15%", "$8.29 every 4 weeks · Save $1.46" and a **Subscribe** button.
- **One tap switches the item.** The whole strip is the button. The cart drawer updates in place. No reload, no product page.
- **Subscribed items keep the strip**, now showing a frequency dropdown built from your plans, "You save $1.46 on every delivery" and a "Switch to one-time" link.
- **The one-time price stays crossed out** on every subscribed item in the cart, whether it was switched in the cart or added as a subscription from the product page.
- **Savings at the cart total.** Shopify treats a subscription as a lower price, not a discount, so themes show no savings at the total. This adds them: what the cart would cost with everything one-time, crossed out next to the total, and "You're saving $1.46 on every delivery with Subscribe & save" under it.
- **Only where it applies.** Products without a subscription plan get nothing.
- **Savings follow the quantity.** Three cases at $1.46 off each shows $4.38.
- **Nothing is hardcoded.** The percent, the prices and the frequencies all come from your selling plans. Change the plan in your subscription app and the cart follows.

Built and tested on Shopify's **Horizon** theme. It's written to work in other Online Store 2.0 themes too (every free Shopify theme since 2021). See [Other themes](#other-themes) if something doesn't line up.

## Before you start

You need a subscription app that uses Shopify's selling plans. **Shopify Subscriptions** (free, made by Shopify) works, and so do most others. Set up a plan with a discount on the products you want to offer. The button only shows when subscribing actually saves the shopper money.

Optional but worth it: add your subscription app's widget to your product page too (in the theme editor: product template, **Add block > Apps**). Shoppers who pick one-time there still see the button in the cart.

## Install (about 3 minutes, no coding)

**Tip:** try it on a copy of your theme first. In **Online Store > Themes**, click **...** next to your theme and choose **Duplicate**. Test on the copy, then publish it.

### Step 1: Add the section file

1. Open [`sections/cart-subscribe-upgrade.liquid`](sections/cart-subscribe-upgrade.liquid) here on GitHub and click the **Copy raw file** button (the two-squares icon at the top right of the file).
2. In your Shopify admin, go to **Online Store > Themes**. Next to your theme, click **...** then **Edit code**.
3. In the file list on the left, find the **sections** folder and click **Add a new section**.
4. Name it `cart-subscribe-upgrade` and pick **Liquid** if it asks.
5. Delete the starter code Shopify puts in the new file, paste in what you copied, and click **Save**.

### Step 2: Turn it on in the theme editor

1. Make sure your theme's cart opens as a **drawer**. In most themes that's **Theme settings > Cart > Cart type: Drawer**.
2. Go to **Online Store > Themes > Customize**.
3. In the left sidebar, scroll to the **Footer** area and click **Add section**. Pick **Cart subscribe upgrade**.
   It doesn't show anything in the footer. It places itself inside the cart drawer.
4. Click **Save**, add a product with a subscription plan to the cart as a one-time purchase, and open the cart.

### Using Shopify CLI instead

Copy `sections/cart-subscribe-upgrade.liquid` into your theme's `sections/` folder and add it to `sections/footer-group.json`:

```json
"cart_subscribe_upgrade": { "type": "cart-subscribe-upgrade", "settings": {} }
```

Add `"cart_subscribe_upgrade"` to the end of the `order` list in the same file, then push to an unpublished theme:

```bash
shopify theme push --unpublished --theme "Subscribe upgrade test"
```

## Settings

Everything below is in the theme editor under **Footer > Cart subscribe upgrade**.

| Setting | What it does |
|---|---|
| Frequency the button picks | Which plan one tap subscribes to, by its position in the product's list. With plans every 2, 4 and 8 weeks, **Second** (the default) picks 4 weeks. The shopper can change it after. |
| Headline | `Subscribe & save [percent]` by default |
| Second line | `[price] every [frequency] · Save [savings]` by default |
| Button label | `Subscribe` by default |
| Outline each item and its offer as one card | On by default. Turn it off if it clashes with your theme's cart; the offer then shows as its own box under the item. |
| Label | `Subscribed` by default |
| Frequency label | `Deliver every` by default, next to the dropdown |
| Savings message | `You save [savings] on every delivery` by default |
| Switch back link | `Switch to one-time` by default |
| Show the one-time price crossed out on subscribed items | Crosses out the one-time price next to the theme's own price for that item. On by default. |
| Hide the theme's plan name under subscribed items | Many themes print the plan name ("Deliver every 4 weeks, 15% off") under the item. The dropdown already says it, so this hides the duplicate. On by default. |
| Show subscription savings at the cart total | Crossed-out one-time total plus the savings line. On by default. |
| Savings line | `You're saving [savings] on every delivery with Subscribe & save` by default. `[savings]` is the total across every subscribed item. |
| While updating / If the update fails | Status messages on the button |
| Colors | Subscribe button (bottle green by default), button text, savings highlight (yellow), offer strip background, item card outline, headline/checkmark/savings color |
| Theme compatibility (advanced) | Leave these blank. See [Other themes](#other-themes). |

Placeholders work in every message: `[percent]` is the discount, `[price]` the subscription price, `[savings]` the savings per delivery for that item and quantity, `[frequency]` the delivery frequency (like "4 weeks").

Every word a shopper sees is a setting, so you can translate it or change the tone without touching code.

## Discounts on subscribed items

Shopify discounts have a purchase type: one-time, subscription, or both. A discount set to one-time only drops off when the shopper switches an item to a subscription, so their cart total can move by more than the subscription savings. If you want your other discounts to stack with Subscribe & Save, edit each one in **Discounts** and set it to apply to both.

The button only ever shows the subscription's own savings: the one-time price minus the subscription price, per delivery.

## Other themes

The section finds your theme's cart drawer, each cart line and its price on its own. It knows Horizon's markup and the common names used in Dawn and most other themes. If your theme names things differently, one of these will happen:

| What you see | Fix |
|---|---|
| No button in the cart drawer | Fill in **Cart drawer selector** with your drawer's element, like `#CartDrawer`. If that's right and there's still nothing, fill in **Cart line row selector** with the element that wraps one cart line. |
| The item switches but the drawer doesn't show it until you reopen the cart | Your theme doesn't listen for Shopify's standard cart events. An AI agent can add your theme's refresh call. See [AGENTS.md](AGENTS.md). |
| No crossed-out price next to the item's price | Fill in **Line price selector** with the element that holds the item's total price. If none is found, the crossed-out price shows inside the offer strip instead. |
| The per-item price isn't crossed out | Fill in **Unit price selector** |
| No crossed-out total or savings line at the cart total | Fill in **Cart total value selector** with the element that holds the total amount. If none is found, the savings line shows under the cart items instead. |

To find a selector: right-click the element on your storefront, choose **Inspect**, and copy its `id` (put `#` in front) or one of its classes (put `.` in front). Or skip that and ask your AI agent with a screenshot of the cart drawer.

## Customize it with Claude

The code is commented and lives in one file. You can change it with [Claude Code](https://claude.com/claude-code), or paste the file into Claude and paste the answer back into **Edit code**. Prompts that work:

**Change the offer**
> Make the button pick the 8-week plan and change the second line to "Ships every [frequency]. You keep [savings]."

> Only show the button on products tagged `subscribe`.

**Change the look**
> Make the button match my theme's primary button colors and corner radius.

> Show the subscribed strip as one line on desktop.

**Add to it**
> Add a "Subscribe to everything" button under the last item when two or more items can subscribe.

> Also show the button on the /cart page, not just the drawer.

**Fit a different theme**
> I installed cart-subscribe-upgrade on the [theme name] theme. The item switches to a subscription but the cart drawer doesn't update until I reopen it. Make it refresh my theme's cart drawer after the switch.

Tell Claude which theme you're on and exactly what you see. A screenshot helps.

## License

MIT. Use it on your store or your clients' stores, change it, ship it.
