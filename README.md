# WooCommerce Frontend SKU + Attribute Search

Custom WooCommerce snippet that extends the **native product search** to match **SKUs** (products + variations) and **attribute term names**.  
- Keeps native relevance while **OR-adding** SKU/attribute matches.  
- Maps **variation SKUs** back to the **parent product**.  
- Searches **global attribute** terms (e.g., `pa_color`, `pa_size`, or a custom `pa_sku`).  
- Frontend only; no changes to admin search.  
- Works with Code Snippets or your child theme’s `functions.php`.  

---

## Working Sample
[Sansefuria.com — search for `LCSAN03`](https://sansefuria.com/?s=LCSAN03)

---

## Features

- Matches `_sku` on **simple products** and **product variations**.
- Fuzzy match on **attribute term names** across all registered `pa_*` taxonomies.
- Preserves native search results and **OR-includes** matched product IDs.
- Variation hits are **resolved to the parent product** for display consistency.
- Scoped query flag prevents side effects in unrelated queries.

---

## Requirements

- WordPress **5.8+** (tested through **6.7.x**, Aug 2025).
- WooCommerce **4.x+** (tested through **10.1.x**, Aug 2025).
- One of the following:
  - [Code Snippets plugin](https://wordpress.org/plugins/code-snippets/) (recommended)
  - Or access to your child theme’s `functions.php`

---

## Installation

### Option 1: Using Code Snippets (Recommended)

1. Install and activate **Code Snippets**.  
   *Dashboard → Plugins → Add New → Search “Code Snippets”*
2. Go to **Snippets → Add New**.
3. Name it: `WooCommerce Frontend SKU + Attribute Search`.
4. Paste the PHP code from your `sku-attribute-search.php`.
5. Set **Run snippet everywhere**.
6. Save and **Activate**.

### Option 2: Add to `functions.php` (Child Theme Only)

1. Open your child theme’s `functions.php`.
2. Paste the PHP code at the **end** of the file.
3. Save the file and clear any caches.

---

## Usage

1. Use the **native site search** or the **shop/category/tag** search field.
2. Enter any of the following:
   - A **SKU** (full or partial) — matches products and variations.
   - An **attribute value** (e.g., a color or size term name).
   - Regular **keywords** (titles, content) — native behavior remains.
3. Results will include native matches **plus** any products matched by SKU or attribute term.  
   *(Variation matches are surfaced as their parent product.)*

---

## Notes & Compliance

- **Hooks**: Uses `pre_get_posts` to collect matches and a guarded `posts_where` filter that activates **only** when the query carries a custom flag (`jsd_psa_or_ids`).  
- **Scope**: WHERE clause OR-adds only **published products** to avoid leaking other post types/statuses.  
- **Attributes**: Searches all registered global attribute taxonomies returned by `wc_get_attribute_taxonomies()` (prefixed `pa_`).  
- **Safety**: No admin impact; frontend main query only; avoids template overrides.

---

## Code (Reference Header)

```php
<?php
/**
 * JSD — WooCommerce search: match product & variation SKUs + attribute term names
 *
 * Behavior:
 * - Keeps native search results intact, and OR-adds products whose SKU (simple/variation) or
 *   attribute term names fuzzy-match the search text.
 * - Frontend only; main query only; product archives + site search.
 *
 * Requirements: WordPress 5.8+ (tested through 6.7.x), WooCommerce 4.x+ (tested through 10.1.x)
 *
 * @author  jsohnData
 * @version 1.1.0 (2025-08-21)
 */

if ( ! defined( 'ABSPATH' ) ) { exit; }

/**
 * Entry: expand the main frontend product/search query with our extra matches.
 */
add_action( 'pre_get_posts', 'jsd_psa_expand_search', 10 );
function jsd_psa_expand_search( $q ) {
	if ( is_admin() || ! ( $q instanceof WP_Query ) || ! $q->is_main_query() ) {
		return;
	}
	if ( ! jsd_psa_is_product_searchish( $q ) ) {
		return;
	}

	$term_raw = (string) $q->get( 's' );
	if ( $term_raw === '' ) {
		return;
	}

	$matched_ids = jsd_psa_collect_matches( $term_raw );
	if ( $matched_ids ) {
		// Ensure we search products on the front end
		$q->set( 'post_type', array( 'product' ) );

		// Flag this query for the WHERE-OR injection (keeps scope tight & removable).
		$q->set( 'jsd_psa_or_ids', $matched_ids );
	}
}

/**
 * Only treat as "searchish" when appropriate.
 * - Native site search
 * - Product archives (shop / category / tag / attribute tax) when an 's' param is present
 */
function jsd_psa_is_product_searchish( WP_Query $q ) : bool {
	if ( $q->is_search() ) {
		return true;
	}
	if ( function_exists( 'is_shop' ) ) {
		if ( ( is_shop() || is_product_taxonomy() || is_product_category() || is_product_tag() ) && $q->get( 's' ) ) {
			return true;
		}
	}
	return false;
}

/**
 * Collect product IDs matched by: product SKU, variation SKU (mapped to parent), and attribute term names.
 */
function jsd_psa_collect_matches( string $term_raw ) : array {
	global $wpdb;

	$like = '%' . $wpdb->esc_like( $term_raw ) . '%';

	// 1) Direct product SKUs
	$product_ids_from_sku = $wpdb->get_col( $wpdb->prepare(
		"SELECT pm.post_id
		 FROM {$wpdb->postmeta} pm
		 INNER JOIN {$wpdb->posts} p ON p.ID = pm.post_id
		 WHERE pm.meta_key = '_sku'
		   AND pm.meta_value LIKE %s
		   AND p.post_type = 'product'
		   AND p.post_status = 'publish'",
		$like
	) );

	// 2) Variation SKUs -> parent product
	$parent_ids_from_variation_sku = $wpdb->get_col( $wpdb->prepare(
		"SELECT DISTINCT p.post_parent
		 FROM {$wpdb->postmeta} pm
		 INNER JOIN {$wpdb->posts} p ON p.ID = pm.post_id
		 WHERE pm.meta_key = '_sku'
		   AND pm.meta_value LIKE %s
		   AND p.post_type = 'product_variation'
		   AND p.post_status IN ('publish','private')",
		$like
	) );

	// 3) Attribute term names (global pa_* taxonomies), including variation-level assignments mapped to parents
	$attr_term_product_ids = jsd_psa_product_ids_from_attribute_terms( $term_raw );

	return jsd_psa_unique_ints( array_merge(
		(array) $product_ids_from_sku,
		(array) $parent_ids_from_variation_sku,
		(array) $attr_term_product_ids
	) );
}

/**
 * Resolve products linked to attribute term names that fuzzy-match the search string.
 * Includes variation term relationships; maps to parent product IDs.
 */
function jsd_psa_product_ids_from_attribute_terms( string $term_raw ) : array {
	if ( ! function_exists( 'wc_get_attribute_taxonomies' ) ) {
		return array();
	}

	global $wpdb;

	$taxes = wc_get_attribute_taxonomies();
	if ( empty( $taxes ) ) {
		return array();
	}

	$parts  = array();
	$params = array();

	foreach ( $taxes as $tax ) {
		$taxonomy = 'pa_' . $tax->attribute_name;

		// Find terms in this taxonomy whose name matches the search
		$found_terms = get_terms( array(
			'taxonomy'   => $taxonomy,
			'hide_empty' => false,
			'search'     => $term_raw,
			'number'     => 200, // safety cap
		) );

		if ( is_wp_error( $found_terms ) || empty( $found_terms ) ) {
			continue;
		}

		$term_ids      = wp_list_pluck( $found_terms, 'term_id' );
		$placeholders  = implode( ',', array_fill( 0, count( $term_ids ), '%d' ) );

		// Build a disjunct for this taxonomy
		$parts[] = "( tt.taxonomy = %s AND tt.term_id IN ($placeholders) )";
		$params[] = $taxonomy;
		foreach ( $term_ids as $tid ) { $params[] = (int) $tid; }
	}

	if ( empty( $parts ) ) {
		return array();
	}

	// Map variation object_ids up to their parent product IDs
	$sql = "
		SELECT DISTINCT
			CASE WHEN p.post_type = 'product_variation' THEN p.post_parent ELSE p.ID END AS product_id
		FROM {$wpdb->term_relationships} tr
		INNER JOIN {$wpdb->term_taxonomy} tt
			ON tt.term_taxonomy_id = tr.term_taxonomy_id
		INNER JOIN {$wpdb->posts} p
			ON p.ID = tr.object_id
		WHERE " . implode( ' OR ', $parts );

	$prepared = $wpdb->prepare( $sql, $params );
	$ids      = $wpdb->get_col( $prepared );

	return jsd_psa_unique_ints( $ids );
}

/** Small helper: sanitize to unique positive ints */
function jsd_psa_unique_ints( array $ids ) : array {
	$ids = array_filter( array_map( 'absint', $ids ) );
	return $ids ? array_values( array_unique( $ids ) ) : array();
}

/**
 * WHERE clause injection: OR-in our matched product IDs.
 * Scope is limited to queries we flagged via 'jsd_psa_or_ids'.
 */
add_filter( 'posts_where', 'jsd_psa_or_matched_ids', 20, 2 );
function jsd_psa_or_matched_ids( $where, $query ) {
	if ( ! ( $query instanceof WP_Query ) ) {
		return $where;
	}

	$ids = (array) $query->get( 'jsd_psa_or_ids' );
	if ( empty( $ids ) ) {
		return $where;
	}

	global $wpdb;
	$ids_sql = implode( ',', array_map( 'absint', $ids ) );
	if ( ! $ids_sql ) {
		return $where;
	}

	// Keep constraints sane: restrict OR-clause to published products
	$where .= " OR ({$wpdb->posts}.ID IN ($ids_sql) AND {$wpdb->posts}.post_type = 'product' AND {$wpdb->posts}.post_status = 'publish')";

	return $where;
}

```
