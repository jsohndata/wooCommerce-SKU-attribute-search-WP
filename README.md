# WooCommerce: Single Product — Add “Smart Contract” Button (WPCode-Compatible)

🤖 **AINF (GPT-5)** — Lightweight, standards-compliant snippet that adds a **“Smart Contract”** button on WooCommerce single product pages.  
The button opens a per‑product contract URL (e.g., Etherscan, Solana Explorer, or a docs page) pulled from product meta.  
Fast, theme‑agnostic, and safe with Astra/Spectra.

---

## Working Sample
[Sansefuria.com](https://sansefuria.com/?s=RBALO00)

---

## About

This repository contains a small, production‑ready snippet that injects a **Smart Contract** button into the product summary area on single product pages.  
It’s designed to be **copy‑pasteable** into **WPCode**, **Code Snippets**, or a child theme’s `functions.php`, without altering templates.

- Keeps product templates untouched (hooks only)
- Lets you control the target URL per product
- Works side‑by‑side with other buttons (e.g., Contact, Share, etc.)

---

## What It Does

- Adds a visually styled **“Smart Contract”** button on the single product page.
- Reads a per‑product URL from a custom field (meta key: `_smart_contract_url`).
- Falls back gracefully (button is hidden) if the URL is not set.
- Uses WooCommerce hooks (`woocommerce_single_product_summary`) for maximum compatibility.

> Tip: Your URL can point to any explorer (EVM, Solana, etc.) or even a documentation page—whatever best represents the asset’s on‑chain reference.

---

## Features

- 🔩 **Drop‑in**: No template overrides; pure hooks
- 🧩 **Theme‑agnostic**: Safe with Astra/Spectra and most builders
- 🧼 **No jQuery required**: Minimal, modern markup
- 🧰 **WPCode/Code Snippets ready**
- 🛡️ **Graceful fallback**: Button appears only when URL exists
- 🧪 **Compatible with product & variation pages** (shows on parent single product view)

---

## Requirements

- **WordPress** 5.8+ (tested through **6.7.x**, Aug 2025)
- **WooCommerce** 4.x+ (tested through **10.1.x**, Aug 2025)
- Optional: **WPCode** or **Code Snippets** plugin for easy management

---

## Installation

### Option A — WPCode (Recommended)
1. Install/activate **WPCode**.
2. Add a new **PHP Snippet**, set to **Auto Insert → Run Everywhere**.
3. Paste the code from `smart-contract-button.php` (or from this README).
4. Save and **Activate**.

### Option B — Code Snippets
1. Install/activate **Code Snippets**.
2. Create a new snippet, paste the PHP, and set it to run **everywhere**.
3. Save and **Activate**.

### Option C — Child Theme
1. Open your child theme’s `functions.php`.
2. Paste the PHP snippet at the end of the file.
3. Save and upload.

---

## Usage

1. Edit a product in WooCommerce.
2. Add the **Smart Contract URL** in **Custom Fields**:
   - **Meta key**: `_smart_contract_url`
   - **Value**: Full URL (e.g., `https://etherscan.io/address/0x...`)
3. Update the product and view the single product page to confirm the button appears.

> If you don’t see **Custom Fields**, enable it in the screen options (top‑right of the product edit screen), or use any preferred custom fields UI plugin/ACF.

---

## Code Standards & Compliance

- **Hooks only**: Uses `add_action( 'woocommerce_single_product_summary', ... )` with a **stable priority**.
- **Escaping & URLs**: All URLs and attributes are escaped (`esc_url`, `esc_attr`) to prevent injection.
- **Conditional rendering**: Button only renders when the meta value is a valid URL.
- **Compatibility**: Verified behavior on WordPress **6.7.x** and WooCommerce **10.1.x** as of **Aug 21, 2025**.
- **No admin impact**: Frontend‑only output; does not alter admin queries or screens.
- **Reasoning lineage**: Mirrors the safety posture outlined in our recent SKU/attribute search refactor
  (scoped hooks, defensive checks, and theme‑agnostic behavior).

---

## Example Snippet (PHP)

```php
/**
 * WooCommerce: Add “Smart Contract” button to single product pages
 * - Reads `_smart_contract_url` product meta
 * - Hides button if URL missing/invalid
 */
add_action( 'woocommerce_single_product_summary', function () {
    if ( ! function_exists( 'is_product' ) || ! is_product() ) {
        return;
    }

    global $product;
    if ( ! $product instanceof WC_Product ) {
        return;
    }

    $url = get_post_meta( $product->get_id(), '_smart_contract_url', true );
    $url = is_string( $url ) ? trim( $url ) : '';

    if ( empty( $url ) || ! filter_var( $url, FILTER_VALIDATE_URL ) ) {
        return; // no output if no valid URL
    }

    $label = __( 'Smart Contract', 'your-textdomain' );
    $title = sprintf( __( 'Open contract for %s', 'your-textdomain' ), $product->get_name() );

    printf(
        '<p class="jsd-smart-contract-wrap"><a class="button alt jsd-smart-contract-btn" href="%1$s" target="_blank" rel="noopener nofollow" title="%2$s">%3$s</a></p>',
        esc_url( $url ),
        esc_attr( $title ),
        esc_html( $label )
    );
}, 31 );
```

**Styling (optional):**

```css
/* Simple, theme-agnostic touch-up */
.jsd-smart-contract-wrap { margin-top: .75rem; }
.jsd-smart-contract-btn { text-decoration: none; }
```

---

## Security Notes

- Button link uses `rel="noopener nofollow"` and opens in a new tab.
- No user input is stored; you manually set a trusted URL.
- Always validate and sanitize any data you store in `_smart_contract_url`.

---

## Changelog

- **1.0.0** — Initial release.

---

## License

MIT (or your preferred OSS license). See `LICENSE` if included.
