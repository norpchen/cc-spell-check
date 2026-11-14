# cc-spell-check

Live spell check for the Claude Code terminal. The desktop app has spell check in its input box; the terminal TUI does not — this plugin adds it.

> Note: Claude Code also ships a native `spellcheck` setting (underline-only, off by default):
> `{ "spellcheck": { "enabled": true, "checker": "aspell", "language": "en_AU" } }` in `~/.claude/settings.json`.
> This plugin goes further: fix-suggestion buttons above the prompt, a persistent personal
> dictionary, and a `/spell` command. Enable one or the other, not both (double underlines).

- Red underlines on misspelled words as you type, painted directly in the prompt box (`prompt.edit` decorations).
- A suggestion band above the prompt with fix buttons and an add-to-dictionary button.
- `/spell` command: `on | off | status | list | add <word> | remove <word> | fix <typo> <fix> | unfix <typo>`.
- Dictionary from `aspell dump master` (default lang `en_AU`, configurable in `/config`), with a hunspell wordlist fallback. Personal dictionary persisted across sessions.
- The submitted prompt is never rewritten — spell check is visual only.

## Local fork changes

This copy differs from upstream (`0xJonty/cc-spell-check`, base commit `1d12f61`). A `git pull` will conflict, so re-apply these after any update.

**Keyboard chords.** The band's buttons only had hotkeys that work after focusing the band (`ctrl+x tab`). Each button now also carries an `action` prop, so a chord works from the prompt. The engine has no plugin-owned chords, so the buttons borrow built-in actions that do nothing while typing. Bind them in `~/.claude/keybindings.json`, `Global` context:

| Chord | Borrowed action | Does |
|---|---|---|
| `alt+1` | `plugin:favorite` | Apply suggestion 1 |
| `alt+2` | `plugin:install` | Apply suggestion 2 |
| `alt+3` | `plugin:toggle` | Apply suggestion 3 |
| `alt+0` | `plugin:cycleMarketplace` | Add the word to the dictionary |

```json
{ "context": "Global", "bindings": {
  "alt+1": "plugin:favorite", "alt+2": "plugin:install",
  "alt+3": "plugin:toggle",   "alt+0": "plugin:cycleMarketplace" } }
```

While the plugin manager is open, these chords also act on its own actions. Suggestions 4 to 9 have no chord. They use the focus route.

**Better suggestions.** Candidates at the same edit distance rank in this order:
1. Swapped adjacent letters first (`teh` gives `the`).
2. More common words first, from `hooks/frequency.ts`.
3. Then the upstream rules: same first letter, closest length, alphabetical.

`hooks/frequency.ts` holds the 10,000 most common English words, from [aparrish/wordfreq-en-25000](https://github.com/aparrish/wordfreq-en-25000), an export of the [wordfreq](https://github.com/rspeer/wordfreq) project. License: CC BY-SA 4.0.

**Your own fixes.** `/spell fix recieve receive` always puts `receive` first for `recieve`. `/spell unfix recieve` removes it. `/spell list` shows the dictionary and the fixes. Both live in the plugin's store file, `~/.claude/plugins/store/cc-spell-check_inline-*.json`, under `personalWords` and `suggestOverrides`.

**Setting.** `maxSuggestions` (in `/config`, "Suggestions offered"): 1 to 9, default 3.

**Windows.** aspell output with CRLF line endings is split correctly, so the dictionary loads.

**Tests.** `claude plugin test .` runs 25 tests, including a regression test for a crowded dictionary. The upstream `teh` test used a nine-word dictionary and missed the ranking bug. One gotcha: keep the dictionary load in `session.start` to a single `await` (the two store reads share one `Promise.all`). An extra step makes `hooks.test.ts` fail with a late `$.state.get refused`.

## Requirements

- Claude Code >= 2.1.x (function-hooks plugin API)
- `aspell` + an aspell dictionary (`sudo apt install aspell aspell-en`)

## Install

Add to `~/.claude/settings.json`:

```json
{ "env": { "CLAUDE_CODE_PLUGIN_DIRS": "/home/jonty/claude-plugins/cc-spell-check" } }
```

or launch with `claude --plugin-dir /home/jonty/claude-plugins/cc-spell-check`.

## Development

```sh
claude plugin validate .
claude plugin test .
npx -y -p typescript tsc -p . --noEmit   # after first load has generated .claude-plugin/types/
# (bare `npx tsc` resolves to a decoy npm package named "tsc" — always pin -p typescript)
```
