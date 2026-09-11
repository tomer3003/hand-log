# Hand Log

A single-page poker hand recorder. Set the seats and the button, deal the cards, and watch
each player's equity move street by street. Hands can be labelled, dated and saved.

No build step, no dependencies, no server — one HTML file.

## Using it

- **Seats** — 2 to 9. Each pod has a **Dealer** button that moves the button and re-labels
  every position around the table. **Fold** removes a player from the equity maths.
- **Cards** — click any slot, on the felt or in a pod, to pick a card. Cards already in play
  are greyed out. Player names are editable in place, and renaming is retroactive — every
  action already logged, on every street, takes the new name.
- **Card picker style** — a toggle in the picker header switches between showing all 52
  cards at once and a two-step flow: choose the rank, then the suit. The two-step layout has
  much larger targets, which suits a phone. The choice is remembered.
- **Blinds** — set the stakes in the header. They are saved with the hand and remembered for
  the next one, and they price a preflop call.
- **Action** — log what each player did on each street, as many actions per street as the
  hand needs (check, then re-raise after someone raised, and so on). Bet sizes are optional.
  The form follows the table: players are listed in the order they act — under the gun first
  preflop, the small blind first on every street after — and it starts on whoever is next to
  act, moving round as you add each one. The action defaults to **Call** with the amount
  already filled in (the largest bet or raise on that street, or the big blind preflop), or
  **Check** when there is nothing to call, which is what the big blind gets when nobody has
  raised, and what everyone gets once the betting has come back round to whoever raised last,
  since the street is then settled. All of it is a suggestion: pick any player, any action,
  type over any amount.
  Click any logged action to jump to that moment, **↑ ↓** to reorder one within its street,
  **Edit** to correct one in place — player, action or size, and it stays where it is — or
  **Remove** to drop it.
- **Run hand** replays the hand move by move: it deals each street, then plays that street's
  actions one at a time, naming the player on the felt and highlighting their seat. The
  street buttons jump to any point and the left/right arrow keys step one action at a time.
  Undealt cards stay face-down and later streets stay blank, so a replay doesn't spoil it.
- **Equity** is the share of the pot each player expects to win, split pots counted as a
  share, measured at the start of each street. A logged fold takes that player out from the
  next street on; if it leaves one player, the pot shows as uncontested. The **In/Out**
  toggle on a pod is separate — it means the player was never in the hand being modelled. Flop, turn and river are exact — every remaining runout is enumerated. Preflop is
  sampled over 40,000 runouts and is accurate to roughly ±0.3%. Leave a player's cards blank
  and they're treated as holding a random hand.
- **Save hand** logs it under the **Played** date, which you can set to any past date and
  edit later from the list. The log groups by day, newest first.
- **Export / Import** in the saved-hands panel moves your whole log between browsers as
  JSON, and doubles as a backup.

## Where hands are stored

In the browser's own storage, on the device you saved them on. They are not uploaded
anywhere. Clearing site data will erase them, and a log saved on your phone will not appear
on your laptop — use **Export** and **Import** to move between the two.

## Hosting it

`index.html` is the whole app, so any static host works. For GitHub Pages: put this repo on
GitHub, then in **Settings → Pages** set the source to **Deploy from a branch**, branch
`main`, folder `/ (root)`. The site appears at
`https://<your-username>.github.io/<repo-name>/` within a minute or so.

Because it needs no server, you can equally open `index.html` straight from disk — though
note that iOS Quick Look and other in-app file previews block JavaScript, which leaves the
page looking empty. Open it in a real browser.

## Accuracy

The hand evaluator has been checked against exhaustive enumeration: heads-up AA vs KK
returns 81.26% over all 1,712,304 boards, matching published figures.
