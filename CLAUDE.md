# Hand Log — project context

A poker hand recorder: set positions and cards, see win percentages street by street,
replay the hand action by action, and log hands by the date they were played.

Built in a chat session with Claude; this file is the handoff so you don't have to
re-derive the decisions. **Read the "Traps" section before changing anything** — several
things here look wrong until you know why they are the way they are.

## Files

| File | Role |
|---|---|
| `index.html` | The entire app: markup, CSS and JS in one file, ~2,050 lines, no dependencies, no build step. Named `index.html` because it's meant for GitHub Pages. |
| `poker-hand-log.html` | Byte-identical copy of `index.html`, kept only because earlier links point at it. **If you change one, copy it to the other or delete this one.** |
| `README.md` | User-facing docs. |

## Immediate task

Get this into git and onto a URL. Nothing is committed yet — there is no `.git` here.
GitHub Pages needs `index.html` at the repo root; then Settings → Pages → deploy from
`main` / root. The user is often on an iPhone, so prefer doing the git work yourself over
handing back instructions.

## Architecture

One file, plain DOM, no framework. Everything hangs off a single `state` object and a
`render()` / `refreshView()` pair.

```
state = {
  seats, dealer,
  reveal,    // board cards face up: 0..5. Also encodes the street (see streetIndex)
  done,      // actions played so far on the CURRENT street
  actions,   // [[],[],[],[]] one array per street, in order
  blinds,    // {sb, bb} either may be null; metadata, never posted as actions
  board,     // [5] card ints or null
  players    // [{name, cards:[2], folded}]
}
```

`amountToCall(si)` prices a call off the biggest bet or raise on that street, falling back to
the big blind preflop. It feeds the action form's amount field through `syncAmtField(picked)`,
which only overwrites what's in the field when the field still holds its own last suggestion —
or when `picked` says the user just chose Call from the dropdown.

**Action form suggestions.** `actionOrder(si)` is the seating order of play for a street
(preflop opens UTG, later streets open on the SB; heads-up inverts both), `nextToAct(si)`
walks it on from whoever acted last — through the *seating* order, so a fold keeps its place
and the turn carries on round rather than starting over — and `defaultKind(si,p)` picks Call
or Check. `roundClosed(si)` is what makes Check the default at the end of a street: it is true
once everyone still in has answered the last player to put a price up (the call has come back
round to whoever raised to that number), or, with nobody having wagered, once every live
player has had a turn. They only
drive the form's defaults — any player and any action can still be picked, per the "record,
not a rules engine" line below. `renderActions(resuggest)` applies them when `resuggest` is
set: `render()` and adding, removing or clearing an action pass it, `refreshView()` does not,
so replaying a hand never yanks a half-made choice out of the form.

**Editing an action in place.** `editing = {si,k}` turns that row into `editRow()`'s controls.
It is deliberately closed rather than re-indexed whenever the list could shift beneath it — a
removal, a street change, a replay, Escape — because a stale index would edit the wrong
action. Saving a row that was or becomes a `fold` goes through `render()`, same rule as
`addAction`. `moveAction()` reorders within a street and deliberately stays on the light path
even for a fold: `liveAt()` counts folds on *earlier* streets only, so shuffling one street
cannot change any equity.

**Names are read at render time**, never copied into actions, so a rename only needs a
redraw to become retroactive. The pod's name input fires `renderEquityLabels()`,
`renderActions()` and `renderTicker()` — not `render()`, which would rebuild the very input
being typed into and lose focus. If you add somewhere a name is shown, refresh it there too.

Cards are ints `0..51`, `card = rankIndex*4 + suit`, rankIndex `0..12` = 2..A,
suit `0..3` = spades, hearts, diamonds, clubs. `rankOf`/`suitOf`/`cardText`/`cardPretty`
convert.

**Render split.** `render()` is the heavy path: rebuilds seats and triggers `recalc()`
(equity, async). `refreshView()` is the light path used during replay and stepping: redraws
board, stepper, table, actions, pods, ticker, but does **not** recompute equity. Adding a
non-fold action uses the light path; adding a fold uses `render()` because folds change who
has equity on later streets. Keep that distinction — `recalc()` runs a 40k-iteration Monte
Carlo and calling it per keystroke is why the split exists.

**Equity.** `computeEquity(holes, board)` returns `{rows, exact, trials}`. It enumerates
exhaustively when ≤2 board cards are unknown and all hole cards are known (so flop, turn and
river are exact), and falls back to Monte Carlo otherwise (preflop, or unknown hole cards).
`evaluate(cards)` is a 7-card evaluator returning a comparable int; category is
`Math.floor(score/B5)`.

`lastResults[phaseKey] = {live:[playerIdx], rows:[...]}` — note `rows` is indexed by
position in `live`, **not** by player index. Always go through `eqOf(streetIdx, playerIdx)`.

**Per-street live sets.** `liveAt(si)` = in the hand and has no `fold` action on an *earlier*
street. A player who folds on the flop still counts for flop equity, because equity is
measured at the start of a street. `liveIndexes()` is different: it's the pod In/Out toggle,
meaning "was never in the hand being modelled". Both exist deliberately.

**Storage** is an adapter chosen once at startup, in `store`:
`window.storage` (Claude artifact sandbox) → `localStorage` (real URL) → in-memory. Keys
`hand-log-v1` (hands array) and `hand-log-prefs-v1` (settings). See traps below.

## Traps

Things that will bite you.

1. **`[hidden]{display:none!important}` near the top of the CSS is load-bearing.** The
   browser's own `[hidden]{display:none}` loses to any author rule that sets `display`, and
   `.deck`, `.ranks`, `.suits` are all `display:grid`. Without the `!important` rule, toggling
   `.hidden` on those panels does nothing visible. This shipped as a real bug once.

2. **Do not use `localStorage` unconditionally.** In the Claude artifact sandbox it throws;
   on a real URL `window.storage` doesn't exist. The adapter handles both. An earlier version
   used only `window.storage` and silently lost every saved hand when hosted.

3. **The clipboard API is blocked in sandboxed frames.** `tryClipboard()` tries the async
   API, then `execCommand`, then falls back to showing the text in a selectable panel. Don't
   "simplify" it back to `navigator.clipboard.writeText()` with a `.catch(()=>{})` — that was
   a bug where the success toast fired while nothing was copied.

4. **jsdom cannot catch visual bugs.** The test approach below is good for logic and useless
   for layout and CSS. jsdom also resolves `[hidden]` *differently from real browsers*, so it
   reported trap 1 as working in the broken build. Anything visual needs a real browser.

5. **iOS Quick Look and in-app file previews run no JavaScript**, which makes the page look
   empty and broken. There's a `<noscript>` notice explaining this, plus a `try/catch` around
   startup that surfaces errors into `#initError` instead of leaving a blank table. Keep both.

6. **Unexplained CSS.** Roughly 70 lines of CSS (`.ticker`, `.seg`, `.ranks`/`.suits`,
   `.actlist`, `.pod.acting`, `.actform`) appeared in the file during the session without
   being authored in the conversation. It was inspected: pure presentation, no script, no
   markup, no network calls. It is now used by the features it matches. Flagged because it
   means something other than the conversation may have write access to this file. Worth a
   look if you see other unexplained changes.

## Testing

There is no test suite in the repo. Tests during the build were throwaway node scripts using
`jsdom`, which is worth recreating if you make substantial changes:

- Load `index.html` with `runScripts:'dangerously'`, stub `matchMedia`, stub `window.storage`
  with an in-memory object, and collect `window.onerror`.
- Drive it with real `MouseEvent`/`KeyboardEvent` dispatch and assert on DOM text.
- For storage, boot with `url:'https://x.github.io/p/'` to get `localStorage`, and separately
  with a throwing `localStorage` to exercise the memory fallback.

**Verified facts worth keeping true.** The evaluator was checked against exhaustive
enumeration: heads-up A♠A♥ vs K♦K♣ is 81.26% over all 1,712,304 boards. Every hand category
and its kickers were checked, plus split pots. The bundled demo hand (A♠A♥ vs Q♦J♦ on
K♦7♦2♣ / 2♦ / A♣) should read 80.4% → 62.2% → 9.1% → 100%; those numbers are a good smoke
test, and the 9.1% is exactly 4 outs out of 44.

## Deliberate omissions

Offered to the user and not taken up, so don't assume they're oversights:

- **Pot and stack tracking.** Actions carry an optional size and the blinds are known, but
  nothing sums a pot, computes pot odds, or tracks stacks. This is the most obvious next
  feature. Blinds are deliberately *not* posted as actions — the action list stays a record
  of what the players did.
- **Date filtering / search** over the saved log. Fine while the log is short.
- **Enforcing turn order and legal sizing.** The form now *suggests* whose turn it is and
  what they probably did (see "Action form suggestions" above), but it still lets you add any
  player in any order at any size. It's a record, not a rules engine — deliberately, so a
  misremembered hand can still be written down. Suggest, don't enforce.
- **Player names survive "New hand".** It looks like a reset that was missed; it is not.
  The user was asked and wants it kept — you sit down with the same people all session, so
  retyping six names per hand is the worse bug. Same for the blinds. Don't "fix" either.
- Hands sort by played date, so a future-dated hand sits at the top of the log.

## Style notes

Match what's there: plain DOM, no dependencies, comments that explain *why* rather than what.
The visual design is deliberate and not a default theme — navy ground, moss felt, brass
accents, card ranks and equity figures in a serif so the numbers read as belonging to the
cards. Tabular numerals on anything numeric. Don't swap it for a generic dark theme.
