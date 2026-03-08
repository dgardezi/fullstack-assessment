# Stackline Full Stack Assignment

## Bugs Identified

- **Product card "View Details" button should align to the bottom of each card** — Product cards have varying content height due to length of titles and categories. The card uses flex column layout but the middle section doesn’t grow, so the footer (and "View Details" button) doesn’t sit at the bottom of the card and rows look uneven.
  - Added `flex-1 min-h-0` to `CardContent` on the product cards so the content block grows and pushes the footer down.
  - The card is already a flex column with `h-full` but the middle section needed to consume remaining space and `flex-1` accomplishes that. `min-h-0` prevents the flex item from imposing a content-based minimum height so the layout works correctly in the grid.

- **Results count should use singular "product" when only one result** — The copy is hardcoded as `Showing {products.length} products`, so it incorrectly shows "Showing 1 products" when there’s one result.
  - Updated label to conditionally render "product" when only one `products.length === 1` and "products" otherwise.
  - Keeps the count grammatically correct and matches expected UX for result summaries.

- **"Clear Filters" button does not reset the dropdowns to their placeholder** — Clicking "Clear Filters" sets `selectedCategory` and `selectedSubCategory` to `undefined`, but the dropdowns keep showing the previous selection instead of reverting to "All Categories" / "All Subcategories".
  - Added a `key` prop to both Select components derived from the current value (e.g. category-${selectedCategory ?? "none"}) so that when state is cleared the key changes and React remounts the Select, which then forces it to show the placeholder.
  - Remounting avoids relying on Radix Select's behavior when switching from a string value to `undefined` and keeps the UI in sync with state.

- **Subcategories are not filtered by selected category** — When a category is selected, the client calls `GET /api/subcategories` with no query params. The API supports `GET /api/subcategories?category=...` and filters by category, but the client never sends `category`. This means the subcategory dropdown shows all subcategories from all categories instead of only those for the selected category.
  - Updated request URL to pass the selected category as a query param so the API returns only subcategories for that category.
  - Using `encodeURIComponent` keeps the URL safe in case any category names contain special characters.

- **Product detail page can crash on missing or tampered data** — The detail page reads `product` from the URL query and uses `product.featureBullets.length` and `product.retailerSku` without guarding for missing/undefined. If the URL is tampered with or the payload is incomplete, `featureBullets` or `retailerSku` can be undefined and the page will throw an error.
  - Updated to use optional chaining and nullish coalescing so missing or tampered fields render safely instead of throwing uncaught errors.
  - Features section and image thumbnails only render when the corresponding data is present.

- **No error handling on API fetches** — All `fetch()` calls use `.then()` only and no `.catch()`. If a request fails for any reason, loading state can stay true indefinitely or state can be left stale with no user feedback.
  - Added `.catch()` to categories, subcategories, and products fetches. On failure, set loading to false and use empty arrays as a fallback so that we avoid leaving the UI stuck.
  - Check `res.ok` before parsing JSON so non 2xx responses are treated as errors and handled in the catch block.

- **Product detail relies on URL payload instead of the API** — The list page passes the full product object in the URL via `query: { product: JSON.stringify(product) }`, and the detail page parses it and renders it. There are a few issues with this: 1: Users can tamper with the query string and change product data shown. 2: URLs are long and fragile. 3: The existing `GET /api/products/<sku>` route is unused. The detail page should use the product SKU in the URL and fetch the product from the API so the server is the single source of truth and the page is shareable and tamper-resistant.
  - Updated product detail page so it lives at `/product/<sku>`. The list page links to a URL with encodeURIComponent(product.stacklineSku) and no longer passes the product JSON in the query.
  - The detail page reads the SKU from the URL, fetches from `GET /api/products/<sku>`, and shows loading and not found messages on 404 or fetch failure. As a result, the server is the single source of truth. URLs are short and shareable and cannot be tampered with to show fake product data.
  - For backwards compatibility, visiting the old `/product` URL (with no SKU) redirects to the product list (`/`), so old bookmarks or links are not broken.
