# Synia launcher

A single-page launcher for [Synia](https://synia.toolforge.org/). The user chooses a Wikidata aspect page and, depending on that page, no item, one item or two items. The launcher then opens the matching Synia page in a new tab.

It is a static page: one HTML file with inline CSS and JavaScript, no dependencies, no build step, no backend.

The page was generated with an AI assistant from [`SPECIFICATIONS.md`](SPECIFICATIONS.md). Review and test it before relying on it.

## Files

| File | Purpose |
|---|---|
| `synia-launcher.html` | The application (markup, styles, script) |
| `SPECIFICATIONS.md` | Full specification, written to be reused as a generation prompt |
| `how-to-use.html` | User guide without technical details |
| `README.md` | This document |
| `LICENSE` | GPL-2.0 |

## Running it

Host `synia-launcher.html` (and `how-to-use.html` next to it) on any static host: Toolforge static hosting, GitHub Pages, etc. Opening the file locally should also work, since all network calls go to the Wikidata API with `origin=*`, but this has not been tested.

Requirements: a recent browser with `fetch`, `AbortController`, `URL`, `URLSearchParams` and `String.prototype.endsWith`.

## Concepts

- **Aspect page**: a Wikidata page under `Wikidata:Synia:` holding a title and SPARQL queries (for example `Wikidata:Synia:venue`).
- **Short name**: the page title without the `Wikidata:Synia:` prefix (`venue`).
- **Naming convention**: the short name decides how many items the launcher asks for. The test is case-sensitive and applied in this order:

| Short name | Items | Example |
|---|---|---|
| Ends with `-index` | 0 | `library-index` |
| Contains no `-` | 1 | `venue` |
| Contains `-`, does not end with `-index` | 2 | `language-language` |

For two items, the short name is split at the first `-` into `aspect1` and `aspect2`. Each part is paired with one item in the Synia address.

Pages outside the convention (a title that is not under `Wikidata:Synia:`, or a name such as `-foo` with an empty part) get a single optional item field, and the full page title is used as the aspect name.

## Synia addresses

The patterns are constants at the top of the script (`PATTERNS`, `SYNIA_BASE`).

| Case | Pattern |
|---|---|
| 0 items | `https://synia.toolforge.org/#{aspect}` |
| 1 item, filled | `https://synia.toolforge.org/#{aspect}/{qid1}` |
| 1 item, left empty | `https://synia.toolforge.org/#{aspect}` |
| 2 items, both filled | `https://synia.toolforge.org/#{aspect1}/{qid1}/{aspect2}/{qid2}` |
| 2 items, second left empty | `https://synia.toolforge.org/#{aspect1}/{qid1}/{aspect2}` |

Every value is URL-encoded by `fill()`. The last item is always optional; the first item of a two-item aspect is required. The launcher never adds `-index` itself.

Examples:

| Aspect | Items | Result |
|---|---|---|
| `library-index` | none | `https://synia.toolforge.org/#library-index` |
| `venue` | Q1234 | `https://synia.toolforge.org/#venue/Q1234` |
| `language-language` | Q9035, Q150 | `https://synia.toolforge.org/#language/Q9035/language/Q150` |
| `language-language` | Q9035 | `https://synia.toolforge.org/#language/Q9035/language` |

The address with the second item empty is an assumption to confirm against Synia's behaviour.

## Code layout

Everything is in the `<script>` block of `synia-launcher.html`, inside one function scope.

**Pure logic** (between the `// <pure>` and `// </pure>` markers; no DOM, no network, easy to test):

| Function | Role |
|---|---|
| `isQid(text)` | True for `Q` followed by digits (case-insensitive) |
| `requiredItemCount(shortName)` | Applies the naming convention (0, 1 or 2) |
| `parseAspectAddress(raw)` | Validates the address and extracts the page title, or returns `{ error }` |
| `describeAspect(title)` | Returns `kind`, `count`, `aspect` (and `aspect1`/`aspect2`), and the item field definitions with their labels |
| `fill(template, values)` | Replaces `{name}` placeholders with URL-encoded values |
| `buildSyniaUrl(desc, qids)` | Chooses the pattern and builds the Synia address |
| `normalisePages(rawPages)` | Filters, normalises and sorts the pages returned by the API |

**Interface:**

| Area | Functions |
|---|---|
| Storage | `load`, `save`, `getRecent`, `addRecent`, `renderRecent` |
| Item fields | `buildItemField`, `renderItems`, `itemState` |
| Aspect field | `syncAspect`, `applyAspect`, `setAspectMsg` |
| Browse panel | `loadList`, `fetchAspectPages`, `renderList`, `setBrowseOpen`, `choosePage` |
| Submit and start-up | the `submit` handler, URL parameter handling at the end of the script |

## How it works

### Aspect field

`syncAspect()` runs on every input. If the address is valid, `describeAspect()` decides which item fields to show and the button is enabled. If it is invalid, the item fields are removed and the button is disabled. Error messages appear under the field after a half-second pause, on blur and on change. The existence of the Wikidata page is not checked.

Recent addresses are offered through a `<datalist>`.

### Item fields

- Fields are rendered only when their signature (count and labels) changes, so typing in the aspect field does not destroy them.
- Values live in `itemState` and are restored when fields are re-rendered, so the first item survives a change from two fields to one.
- Each field has its own autocomplete. Typing triggers a search after 250 ms:

  ```
  https://www.wikidata.org/w/api.php?action=wbsearchentities&format=json&origin=*
      &type=item&limit=8&language=<lang>&uselang=<lang>&search=<term>
  ```

  `<lang>` is the browser language. A new search aborts the previous one (`AbortController`).
- Text such as `q42` is accepted as a Q-id without a search.
- Keyboard: arrows, Enter and Escape. The list closes shortly after blur; results are chosen on `mousedown` so the blur does not discard the click.

### Browse panel

The list comes from:

```
https://www.wikidata.org/w/api.php?action=query&list=allpages&apnamespace=4&apprefix=Synia%3A
    &apfilterredir=all&aplimit=max&format=json&formatversion=2&origin=*
```

- It is requested the first time the panel opens.
- If the response contains a `continue` object, its keys are added to the next request until the list is complete (limited to 40 requests as a safeguard).
- The raw result (`title` and `redirect` flag per page) is cached in `localStorage` for 24 hours. "Refresh list" bypasses the cache.
- Pages are grouped by the naming convention ("No item", "One item", "Two items", and "Other" if needed) and filtered by short name, ignoring case.
- Redirects carry a "redirect" tag. This relies on the API returning a `redirect` flag; if it does not, no tag is shown.
- Choosing a page fills the aspect field with `https://www.wikidata.org/wiki/<title>` and runs the same logic as a pasted address.

### Submit

1. Validate the aspect address.
2. For each item field: empty and required means "Enter the first item."; text that is not a selected item or a Q-id means "Choose a suggestion or enter a Q-id."
3. Build the Synia address with `buildSyniaUrl()`.
4. Store the aspect address in the recent list.
5. Open the address with `window.open(url, "_blank", "noopener")`. The address is also shown as a link in case the browser blocks the tab.

### URL parameters

| Parameter | Accepted values | Effect |
|---|---|---|
| `aspect` | Full Wikidata address, or a page title such as `Wikidata:Synia:venue` | Fills the aspect field |
| `item1` | Q-id | Fills the first item |
| `item2` | Q-id | Fills the second item |
| `item` | Q-id | Alias of `item1` |

Example: `synia-launcher.html?aspect=Wikidata:Synia:language-language&item1=Q9035&item2=Q150`

Item values are placed in `itemState` before the aspect is evaluated, so they appear in the fields the aspect needs. The form is not submitted automatically.

### Browser storage

All access is wrapped in `try/catch`; the page works without storage.

| Key | Content |
|---|---|
| `synia-launcher-recent-aspects` | JSON array of the last 10 aspect addresses, most recent first, without duplicates |
| `synia-launcher-aspect-list` | `{ fetchedAt, pages }` for the browse list (24-hour validity) |

Older versions of the page stored address patterns under `synia-launcher-template` and `synia-launcher-template-noitem`. They are no longer read and can be ignored.

## Security notes

- Text from the Wikidata API (labels, descriptions, page titles) is inserted with `textContent` or DOM properties, never as HTML.
- External links use `rel="noopener"`.
- The aspect address must be on `wikidata.org` or a subdomain.

## Accessibility and appearance

- Item fields use the combobox and listbox pattern (`role`, `aria-expanded`, `aria-controls`, `aria-selected`).
- Status and selection messages are in `aria-live` regions.
- Keyboard focus is visible on every control.
- Colours are CSS variables on `:root`, with a dark variant under `prefers-color-scheme: dark`.

## Testing

No automated test suite is included. The functions between the `// <pure>` markers can be extracted and run in Node; the following cases are a good starting point:

```js
requiredItemCount("library-index")      // 0
requiredItemCount("venue")              // 1
requiredItemCount("language-language")  // 2
requiredItemCount("library-Index")      // 2 (case-sensitive)
buildSyniaUrl(describeAspect("Wikidata:Synia:language-language"), ["Q9035", "Q150"])
// "https://synia.toolforge.org/#language/Q9035/language/Q150"
buildSyniaUrl(describeAspect("Wikidata:Synia:venue"), [""])
// "https://synia.toolforge.org/#venue"
```

Manual checklist in a browser:

1. Paste `.../Wikidata:Synia:library-index`: no item field appears and the button is enabled.
2. Paste `.../Wikidata:Synia:venue`: one optional field. Search for an item, choose it, open Synia.
3. Paste `.../Wikidata:Synia:language-language`: two fields, the first required. Leave the second empty and check the address.
4. Open the browse panel: groups appear, the filter works, redirects are tagged, choosing a page fills the field.
5. Break the network (offline mode): the browse panel and item search show their error messages and a pasted address still works.
6. Reload with `?aspect=...&item1=...&item2=...`: the form is prefilled.

## Known limitations and assumptions

- The address used when the last item is empty (only its Q-id is dropped, the aspect name stays) is an assumption.
- Names with several dashes are split at the first dash only.
- The `redirect` flag is expected in the `allpages` response.
- Item search covers items only, not properties or lexemes, and uses the browser language.
- Recent addresses are stored per browser, not per Wikidata account.
- Rendered Synia pages have no link back to the launcher; that would require a change in Synia itself.
- The address patterns are not editable in the interface; change the constants at the top of the script.

## License

GPL-2.0. See `LICENSE`.
