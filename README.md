# Synia-launcher
Launcher page for Synia (https://synia.toolforge.org/)

# Synia launcher (prototype)

A single-page launcher for [Synia](https://synia.toolforge.org/). The user picks a Wikidata item (optional) and the URL of an aspect configuration page, and the launcher opens the matching Synia page in a new tab.

It is a static page: one HTML file with inline CSS and JavaScript, no dependencies, no build step, no backend.

## Why it exists

Synia renders an "aspect": a Wikidata page (for example `Wikidata:Synia:work`) holding a title and one or more SPARQL queries. Rendered Synia pages only read that configuration page, so they have no place for an item selector. This launcher sits in front of them and solves three problems:

- finding the Synia page that corresponds to a custom aspect page;
- running the same aspect for a different item;
- sharing a link that carries both the aspect and the item.

## Files

| File | Purpose |
|---|---|
| `synia-launcher.html` | The whole application (markup, styles, script) |
| `README.md` | This document |

## Running it

Host `synia-launcher.html` on any static host (Toolforge static hosting, GitHub Pages, etc.). Opening the file locally in a browser should also work, since the only network call goes to the Wikidata API with `origin=*`, but this has not been tested.

## How it works

### 1. Item field (optional)

- Typing triggers a search after a 250 ms pause (debounce).
- The search calls the Wikidata API:

  ```
  https://www.wikidata.org/w/api.php?action=wbsearchentities&format=json&origin=*
      &type=item&limit=8&language=<lang>&uselang=<lang>&search=<term>
  ```

  `<lang>` is the browser language (`navigator.language`, first two letters). `origin=*` enables cross-origin requests from the browser.
- A new search cancels the previous one (`AbortController`).
- Results appear in a listbox showing label, Q-id and description. The user can click a result or use the keyboard: arrow keys to move, Enter to choose, Escape to close.
- If the typed text matches `Q` followed by digits (for example `q42`), it is accepted as a Q-id directly, without a search.
- The field can be left empty. In that case the page opens the aspect without an item.
- If the field contains text that was never resolved to an item, submitting shows an error asking the user to choose a suggestion, enter a Q-id, or clear the field.

### 2. Aspect field

- Accepts the full URL of a Wikidata page.
- `parseAspect()` validates it:
  - it must be a valid URL;
  - the host must be `wikidata.org` or one of its subdomains;
  - the page title is read from `?title=` if present, otherwise from the `/wiki/<title>` path;
  - underscores and spaces are normalised.
- Two values are derived:
  - `aspect_title`: the full page title, for example `Wikidata:Synia:work`;
  - `aspect` (short name): the title without the `Wikidata:Synia:` prefix, for example `work`. If the title does not start with that prefix, the short name is the full title.
- The page does not check that the Wikidata page actually exists.
- Valid URLs are stored as recent aspects (see "Storage").

### 3. Building the Synia address

Two address patterns are used, depending on whether an item is selected. Both are editable in the "Synia address pattern" section of the page and saved in the browser.

| Case | Default pattern |
|---|---|
| With an item | `https://synia.toolforge.org/#{aspect}/{qid}` |
| Without an item | `https://synia.toolforge.org/{aspect}` |

Placeholders (each value is URL-encoded before substitution):

| Placeholder | Value | Example |
|---|---|---|
| `{qid}` | Selected item (not available in the no-item pattern) | `Q42` |
| `{aspect}` | Short aspect name | `work` |
| `{aspect_title}` | Full page title | `Wikidata:Synia:work` |

The defaults are the current working assumption and should be confirmed against how Synia really builds its addresses. To change them permanently, edit `DEFAULT_TEMPLATE` and `DEFAULT_TEMPLATE_NO_ITEM` at the top of the script.

### 4. Opening Synia

On submit the script validates the inputs, builds the address, saves the patterns and the aspect URL, then calls `window.open(url, "_blank", "noopener")`. The generated address is also shown on the page as a link, which serves as a fallback if the browser blocks the new tab.

### 5. URL parameters

The page can be prefilled from its own address:

| Parameter | Accepted values | Effect |
|---|---|---|
| `item` | A Q-id, for example `Q42` | Fills the item field |
| `aspect` | A full Wikidata URL, or a page title such as `Wikidata:Synia:work` | Fills the aspect field (a title is turned into `https://www.wikidata.org/wiki/<title>`) |

Example: `synia-launcher.html?item=Q42&aspect=Wikidata:Synia:work`

This allows a link placed on a Wikidata configuration page, or shared with colleagues, to open the launcher with the aspect already selected. The form is not submitted automatically.

### 6. Storage

Stored in `localStorage`, per browser, with every access wrapped in `try/catch` so the page still works if storage is unavailable:

| Key | Content |
|---|---|
| `synia-launcher-recent-aspects` | JSON array of the last 10 aspect URLs, most recent first, no duplicates. Shown as `<datalist>` suggestions. |
| `synia-launcher-template` | Saved pattern with an item |
| `synia-launcher-template-noitem` | Saved pattern without an item |

Note: a pattern saved in the browser takes precedence over the default in the code. After changing a default, clear the field or paste the new value once.

## Code layout

Everything is in the `<script>` block of `synia-launcher.html`, in this order:

1. **Settings**: constants (defaults, API address, storage keys, limits).
2. **Storage helpers**: `load`, `save`, `getRecent`, `addRecent`, `renderRecent`.
3. **Item autocomplete**: `search`, `showResults`, `choose`, keyboard handling.
4. **Aspect URL parsing**: `parseAspect`, `buildSyniaUrl`.
5. **Status messages**: `setStatus`.
6. **Submit handler**.
7. **Start-up**: restores patterns and recent URLs, reads URL parameters.

Colours are defined as CSS variables on `:root`, with a dark variant under `prefers-color-scheme: dark`.

## Accessibility

- The item field uses the combobox/listbox pattern (`role`, `aria-expanded`, `aria-controls`, `aria-selected`).
- Status and selection messages are in `aria-live` regions.
- Keyboard focus is visible on all controls.

## Known limitations

- The Synia address patterns are assumptions to confirm.
- The aspect page is not checked for existence.
- Item search covers items only (`type=item`), not properties or lexemes.
- Search labels use the browser language, with no language selector.
- Recent URLs are stored per browser, not per Wikidata account.
- Rendered Synia pages have no "change item" link back to the launcher; that would require a change in Synia itself.
- No automated tests.

## Possible next steps

- Add a language selector for the item search.
- Allow `type=property` and `type=lexeme` in the search.
- Check that the aspect page exists (Wikidata API `action=query&titles=...`) and show a clear error if not.
- Add an optional parameter to submit automatically when both values are provided.
- Add a "change item" link to Synia's rendered pages, pointing to the launcher with the current aspect prefilled.
- Integrate the launcher as a route inside Synia (for example `/launch`) to share the same domain and deployment.
