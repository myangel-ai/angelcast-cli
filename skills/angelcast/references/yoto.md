# Putting episodes on a Yoto player

Yoto's "Make Your Own" (MYO) playlists take uploaded MP3s and play on any Yoto
player linked to the parent's account (a MYO card, or the app's playlist view).
There is no AngelCast-side integration: the agent drives my.yotoplay.com
through the Playwright browser tools, logged in as the parent.

## Prerequisites

- Playwright MCP tools (`mcp__playwright__browser_*`). If they are missing,
  the parent adds the server and restarts the session:
  `claude mcp add playwright -- npx @playwright/mcp@latest`
- A Yoto account. The parent logs in themselves in the Playwright window;
  never type their credentials.
- The episodes already downloaded to a folder (section 4 of SKILL.md).
- `browser_file_upload` only accepts paths under the workspace root, so copy
  each MP3 into `<workspace>/.playwright-mcp/yoto-staging/` first and upload
  that copy. Delete the staging copies when done.

## Flow

1. `browser_navigate` to `https://my.yotoplay.com/cards`. If the page title is
   "Log in to Yoto", tell the parent to log in in that window and wait for
   them (`browser_wait_for` on text "My playlists").
2. Ask which playlist to use, or default to a new one named after the batch
   (for example "AngelCast: Boston Weekend").
   - New: navigate to `https://my.yotoplay.com/cards/new`, click the "edit"
     control under "Playlist Title", type the title, press Enter.
     Yoto rejects an empty playlist ("This playlist is missing required
     content"), so the first upload happens before "Create playlist".
   - Existing: navigate to `https://my.yotoplay.com/cards/<id>`.
3. For each MP3: click "Upload files here" (it opens a file chooser), then
   `browser_file_upload` with the staged path. The row shows "Transcoding..."
   for 30 to 60 seconds; `browser_wait_for` in 30 second steps until the row
   shows a duration instead.
4. Count the tracks with `browser_evaluate`:
   ```js
   () => [...document.querySelectorAll('input[placeholder="Click to edit"]')].map(i => i.value)
   ```
   Expect one more entry than before. Yoto has been seen to add the same
   file twice; if so, delete the extra through that track's "Open menu".
5. Rename the new track from its filename to the episode title. Element refs
   on this page go stale after every re-render, so set the value with
   `browser_evaluate` instead of click-and-type:
   ```js
   () => {
     const inputs = [...document.querySelectorAll('input[placeholder="Click to edit"]')]
       .filter(i => i.offsetParent !== null);
     const t = inputs[inputs.length - 1];
     Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set.call(t, '<EPISODE TITLE>');
     t.dispatchEvent(new Event('input', {bubbles: true}));
     t.dispatchEvent(new Event('change', {bubbles: true}));
     t.dispatchEvent(new Event('blur', {bubbles: true}));
     return inputs.map(i => i.value);
   }
   ```
   Check the returned value; the change shows in the next snapshot.
6. New playlist: click "Create playlist". The page moves to
   `https://my.yotoplay.com/cards/<id>` and shows "Playlist successfully
   created"; that URL is the playlist to reuse for later batches. Existing
   playlist: click "Update playlist" and wait for "Playlist updated".
7. Report the playlist URL, and each track's number, title, and duration.

## Gotchas

- Take a fresh snapshot before every click; never reuse a ref. After a
  rename the "Update playlist" button gets a new ref.
- The "Upload files here" button has no stable selector; find it in a fresh
  snapshot of the "Upload audio" group.
- Upload one file at a time and verify the count after each.
- If the session is interrupted, the browser keeps its state; resume from
  the last confirmed step instead of starting over.
