# Discord Web Spellchecker

A lightweight Chrome (Manifest V3) extension for `discord.com` that forces spellcheck on Discord's Slate.js message composer and provides an instant right-click spelling suggestion menu with a persistent custom dictionary.

## Files

- `manifest.json` — Manifest V3 configuration injecting `content.js` on `https://discord.com/*` and `https://*.discord.com/*` at `document_idle`.
- `content.js` — All spellchecking, suggestion ranking, right-click menu UI, Slate.js text replacement, and `spellcheck="true"` enforcement logic.

## What It Does

### Original Functionality (Built with Grok)
- **Built-in English Dictionary (`COMMON_ENGLISH_WORDS`)**: Embeds a ~10,000-word frequency-ranked English word list (`wordSet` + `wordRanks`) directly in `content.js` for zero-latency offline lookups.
- **Edit-Distance Suggestion Engine (`edits1`, `getSuggestions`)**:
  - Computes Levenshtein edit-distance 1 candidates (deletions, transpositions, substitutions, insertions), and falls back to edit-distance 2 for words up to 10 characters.
  - Ranks up to 5 suggestions by word frequency and preserves original capitalization (`ALL CAPS`, `Title Case`, or `lowercase`).
- **Custom Dictionary (`localStorage`)**:
  - Stores user-added words under `discord_spellcheck_custom_dictionary` in `localStorage`.
  - Adds an **"Add to dictionary"** option at the bottom of the right-click menu.
- **Custom Right-Click Menu (`contextmenu` capture listener)**:
  - Detects when you right-click a misspelled word inside Discord's composer (`[role="textbox"]`, `[contenteditable="true"]`, or `[data-slate-editor="true"]`).
  - Uses `caretRangeFromPoint` / `caretPositionFromPoint` to locate the exact word under the cursor and opens a Discord-themed floating menu.
  - Supports full keyboard navigation (`ArrowUp`, `ArrowDown`, `Enter`, `Tab`, `Escape`) and mouse hover/click selection.

### Fixes & Additions (Nov 2026)
- **Slate.js-Safe Text Replacement (`replaceWordAtRange`)**:
  - Fixed a bug where correcting a word and then pressing Space (or clicking elsewhere and pressing Space) reverted the word back to the original typo.
  - Root cause: clicking a suggestion `<button>` blurred Discord's Slate composer, and calling `document.execCommand("insertText")` directly mutated the DOM without updating Slate's internal document model, causing Slate to revert the leaf on the next reconciliation.
  - Fix:
    1. Added `mousedown` `e.preventDefault()` on the popup menu so clicking a suggestion keeps focus in Discord's composer.
    2. Dispatches a `beforeinput` (`InputEvent`) with `inputType: "insertText"` and explicit `targetRanges: [StaticRange]` so Slate intercepts the edit and updates its internal state cleanly.
    3. Includes a synthetic `paste` (`ClipboardEvent`) fallback (after syncing `selectionchange`) and a final `document.execCommand("insertText")` fallback for non-Slate inputs.
- **Apostrophe Boundary Trimming (`extractWordAtPoint`)**:
  - Adjusts the replacement `Range` start/end offsets when stripping leading/trailing apostrophes so single-quoted words (e.g. `'teh'`) keep their surrounding quotes.
- **Native `spellcheck="true"` Enforcement (`enableSpellcheck`, `MutationObserver`)**:
  - Forces `spellcheck="true"` on Discord's composer on page load, on `focusin`, and via a `MutationObserver` whenever Discord mounts a new composer (switching channels, threads, message edits) or sets `spellcheck="false"`.
- **Post-Correction Spellcheck Re-trigger (`nudgeSpellcheck`)**:
  - After Slate updates the text node, briefly toggles `spellcheck` `false` → `true` on the next animation frame so Chrome immediately re-underlines any remaining typos on the line instead of clearing all underlines until you type again.
- **Keyboard Menu Key Support (`extractWordAtSelection`)**:
  - Detects keyboard-triggered `contextmenu` events (`ContextMenu` key / `Shift+F10` / non-mouse pointer), extracts the word at the current text caret instead of the mouse cursor coordinates, anchors the menu below the word, and pre-selects the first suggestion so `Menu` → `Enter` works cleanly.

## Installation / Reloading

1. Open `chrome://extensions` in Chrome and enable **Developer mode**.
2. Click **Load unpacked** and select this folder (`C:\Users\Zai\Projects\discord-spellcheck`).
3. After editing `content.js`, click the **Reload** icon on the extension card and refresh your Discord tab.
