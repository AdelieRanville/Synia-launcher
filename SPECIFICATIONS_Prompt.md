# Synia launcher: specification

Build a single-page web launcher for [Synia](https://synia.toolforge.org/), following the requirements below.

## 1. Goal

The launcher lets a user choose a Wikidata aspect page and, depending on that page, no item, one item or two items, then opens the matching Synia page in a new tab.

An **aspect page** is a Wikidata page, a subpage of `Wikidata:Synia` (for example `Wikidata:Synia:venue`), that holds a title and one or more SPARQL queries. Synia reads it and displays the results.

The launcher solves three problems: finding the Synia page that corresponds to an aspect page, running the same aspect for different items, and sharing a link that carries both the aspect and the items.

## 2. Technical constraints

- One self-contained HTML file with inline CSS and JavaScript. No external libraries, no build step, no backend.
- Plain JavaScript. All calls go to the Wikidata API from the browser, with `origin=*` added to each request.
- Text coming from the API (labels, descriptions, page titles) is inserted with `textContent`, never as HTML.
- Every access to `localStorage` is wrapped in `try/catch`, and the page works normally when storage is unavailable.
- The Synia page opens with `window.open(url, "_blank", "noopener")`.
- Interface text is in English. The item search uses the browser language (`navigator.language`, first two letters).

## 3. User flow

1. The user chooses an aspect page: by pasting its address, picking a recent address, or browsing the list of existing aspect pages.
2. The launcher reads the page name and shows no item field, one item field, or two item fields.
3. The user fills the item fields. The last field is always optional.
4. The user clicks "Open in Synia" and Synia opens in a new tab.

Until the aspect address is valid, no item field is shown.

## 4. Aspect pages and the naming convention

The number of items depends on the **short name** of the aspect page, which is the page title without the `Wikidata:Synia:` prefix. The test is case-sensitive and is applied in this order:

| Short name | Items | Example address | Fields shown |
|---|---|---|---|
| Ends with `-index` | 0 | `https://www.wikidata.org/wiki/Wikidata:Synia:library-index` | none |
| Contains no `-` | 1 | `https://www.wikidata.org/wiki/Wikidata:Synia:venue` | one, optional |
| Contains `-` but does not end with `-index` | 2 | `https://www.wikidata.org/wiki/Wikidata:Synia:language-language` | two, the second optional |

```js
function requiredItemCount(shortName) {
  if (shortName.endsWith("-index")) return 0;
  if (!shortName.includes("-")) return 1;
  return 2;
}
```

For a two-item aspect, the short name is split at the first `-` into `aspect1` and `aspect2` (for `language-language`: `language` and `language`). Each part is paired with one item in the Synia address, and the order of the items can be significant.

**Pages outside the convention.** If the address points to a page that is not under `Wikidata:Synia:`, the full page title is used as the aspect name and a single optional item field labelled "Item (optional)" is shown.

## 5. Page layout

From top to bottom:

1. Title and a one-sentence description.
2. **Aspect field**, with the recent addresses as suggestions, a "Browse aspect pages" button and the explanation note (section 11).
3. **Browse panel**, hidden until opened (section 7).
4. **Item fields**, shown according to the naming convention (section 6).
5. "Open in Synia" button, enabled once the aspect address is valid.
6. Status area for errors and for the generated link.

## 6. Item fields

**Number and labels**

- 0 items: no field. A short note says "This aspect needs no item."
- 1 item: one field, labelled with the short name and "(optional)", for example "venue (optional)".
- 2 items: two fields labelled with the two parts of the name. The second label ends with "(optional)". If both parts are identical, add "first" and "second" to tell them apart: "language (first)" and "language (second, optional)".
- Outside the convention: "Item (optional)".

When the aspect changes, values already entered are kept where possible (the first item stays when going from two fields to one).

**Required and optional**

- The last field is always optional.
- For a two-item aspect, the first field is required.
- An empty last field removes that item from the Synia address (section 8). Synia may then show the equivalent index page; the launcher does not add `-index` itself.

**Autocomplete (each field works independently)**

- Typing triggers a search after a 250 ms pause, using the Wikidata API:

  ```
  https://www.wikidata.org/w/api.php?action=wbsearchentities&format=json&origin=*
      &type=item&limit=8&language=<lang>&uselang=<lang>&search=<term>
  ```

- A new search cancels the previous one for that field (`AbortController`).
- Results appear in a listbox with label, Q-id and description. Mouse selection and keyboard (arrows, Enter, Escape) both work.
- Text of the form `Q` followed by digits (case-insensitive) is accepted as a Q-id without a search.
- The chosen item is shown under the field as "Selected: label (Q-id)".
- Typing again clears the selection. Text that was never resolved to an item blocks submission (section 10).

## 7. Browsing existing aspect pages

A "Browse aspect pages" button opens a panel listing the aspect pages that exist on Wikidata.

**Data source**

```
https://www.wikidata.org/w/api.php?action=query&list=allpages&apnamespace=4
    &apprefix=Synia%3A&apfilterredir=all&aplimit=max&format=json&formatversion=2&origin=*
```

This lists pages in the Project namespace (shown as `Wikidata:`) whose title starts with `Synia:`. Pages are read from `query.allpages`; each has `pageid`, `ns` and `title` (for example `Wikidata:Synia:venue`). Redirect pages are included and carry a `redirect` flag. If the response contains a `continue` value, request the next batch with `apcontinue` until the list is complete.

**Behaviour**

- The list is requested the first time the panel is opened, not when the page loads.
- It is cached in the browser for 24 hours. A "Refresh list" button reloads it.
- A filter field narrows the list as the user types. The filter works on the short name and ignores case.
- Entries are grouped by the number of items they need ("No item", "One item", "Two items"), using the rule in section 4, and sorted alphabetically within each group.
- Each entry shows its short name. Redirect pages carry a visible "redirect" label.
- Each entry is a button: choosing it fills the aspect field with `https://www.wikidata.org/wiki/<title>` and updates the item fields exactly as if the address had been pasted. A redirect is used as is.
- Each entry also has a small link that opens the page on Wikidata in a new tab.
- The panel shows the number of results, a loading message while the request runs, and an error message if it fails ("The list of aspect pages could not be loaded. You can still paste an address.").
- The panel is fully usable with the keyboard.

## 8. Synia addresses

The address depends on the number of items and on which items are filled. The patterns are constants at the top of the script.

| Case | Pattern |
|---|---|
| 0 items | `https://synia.toolforge.org/#{aspect}` |
| 1 item, filled | `https://synia.toolforge.org/#{aspect}/{qid1}` |
| 1 item, left empty | `https://synia.toolforge.org/#{aspect}` |
| 2 items, both filled | `https://synia.toolforge.org/#{aspect1}/{qid1}/{aspect2}/{qid2}` |
| 2 items, second left empty | `https://synia.toolforge.org/#{aspect1}/{qid1}/{aspect2}` |

Placeholders: `{aspect}` is the short name, `{aspect1}` and `{aspect2}` are the two parts of a two-item name, `{qid1}` and `{qid2}` are the items in field order. Every value is URL-encoded.

Examples:

| Aspect address | Items | Synia address |
|---|---|---|
| `.../Wikidata:Synia:library-index` | none | `https://synia.toolforge.org/#library-index` |
| `.../Wikidata:Synia:venue` | Q1234 | `https://synia.toolforge.org/#venue/Q1234` |
| `.../Wikidata:Synia:language-language` | Q9035, Q150 | `https://synia.toolforge.org/#language/Q9035/language/Q150` |

## 9. URL parameters

The page can be prefilled from its own address. The form is not submitted automatically.

| Parameter | Accepted values | Effect |
|---|---|---|
| `aspect` | Full Wikidata address, or a page title such as `Wikidata:Synia:venue` (turned into `https://www.wikidata.org/wiki/<title>`) | Fills the aspect field |
| `item1` | Q-id | Fills the first item field |
| `item2` | Q-id | Fills the second item field |
| `item` | Q-id | Alias of `item1` |

Item parameters are applied after the aspect has determined which fields exist. Example:
`synia-launcher.html?aspect=Wikidata:Synia:language-language&item1=Q9035&item2=Q150`

## 10. Validation and messages

On submit, check in this order and stop at the first problem, moving focus to the field concerned.

| Situation | Message |
|---|---|
| Aspect field empty | Enter the address of an aspect page. |
| Address not a valid URL | That is not a valid web address. Paste the full Wikidata page URL. |
| Host is not `wikidata.org` or a subdomain | The address must point to a page on wikidata.org. |
| No page title found (read from `/wiki/<title>` or `?title=`) | No page title found in that address. |
| First item of a two-item aspect empty | Enter the first item. |
| Typed text not resolved to an item | Choose a suggestion or enter a Q-id. |
| Item search unavailable | The item search is unavailable right now. You can still type a Q-id. |
| Browser blocked the new tab | Your browser blocked the new tab. Open it here: (link) |

Because the button stays disabled until the aspect address is valid, the first four messages are also shown under the aspect field: after the user stops typing for half a second, when the field loses focus, and when a value is chosen or pasted.

On success, show "Opened in a new tab:" followed by the generated address as a link.

The launcher does not check that the aspect page exists.

## 11. Explanation note

A note under the aspect field reads:

> **What is an aspect page?** It is a page on Wikidata that tells Synia what to display: a title and one or more SPARQL queries. You can find existing aspect pages with the "Browse aspect pages" button or on [Wikidata:Synia](https://www.wikidata.org/wiki/Wikidata:Synia).
>
> **Create your own.** Create a new subpage of Wikidata:Synia on Wikidata, for example `Wikidata:Synia:myaspect`, then paste its address here. The name decides how many items the launcher asks for (names are case-sensitive):
> - no dash, such as `venue`: one item;
> - two names joined by a dash, such as `language-language`: two items;
> - a name ending in `-index`, such as `library-index`: no item.

Links in the note open in a new tab (`rel="noopener"`).

## 12. Browser storage

| Key | Content |
|---|---|
| `synia-launcher-recent-aspects` | JSON array of the last 10 aspect addresses, most recent first, without duplicates. Updated on each successful submit. |
| `synia-launcher-aspect-list` | The list of aspect pages with the time it was fetched (24-hour validity). |

## 13. Accessibility and appearance

- Item fields use the combobox and listbox pattern (`role`, `aria-expanded`, `aria-controls`, `aria-selected`).
- Status and selection messages are in `aria-live` regions.
- Keyboard focus is visible on every control.
- Colours are CSS variables on `:root`, with a dark variant under `prefers-color-scheme: dark`.
- The layout is a single centred column (about 38 rem wide) and works on narrow screens.

## 14. Out of scope

- Checking that a typed aspect address points to an existing page.
- Searching for properties or lexemes.
- A link from Synia's rendered pages back to the launcher.
- Recent addresses tied to a Wikidata account.
- A settings panel for editing the address patterns.

## 15. Assumptions to confirm

1. When the last item is left empty, only its Q-id is removed from the address and the aspect name stays (section 8).
2. For names with several dashes (other than a `-index` ending), the name is split at the first dash only.
3. The API marks redirect pages with a `redirect` flag in the `allpages` response.
