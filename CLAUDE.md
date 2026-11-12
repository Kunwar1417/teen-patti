# Teen Patti Table

A single-file web app (`index.html`) for tracking bets during Teen Patti nights with friends. Used on phones and tablets (iPhone + Android tablet). Will be hosted on GitHub Pages at `https://kunwar1417.github.io/teen-patti/`.

## Ground rules
- Keep it ONE self-contained `index.html` (inline CSS + JS). No build step, no frameworks, no npm, no backend unless explicitly asked.
- Only external loads: Google Fonts (Bricolage Grotesque for display, Manrope for body). Everything else (sounds, icons, confetti) is generated in code.
- Mobile first: test at iPhone width (390px). No horizontal page scroll; only tables may scroll sideways inside their own container.
- Money is whole rupees (integers). Every round's nets must sum to exactly ₹0. Never break this.
- Never use em dashes in any user-facing text.
- Plain, friendly English in the UI. Say "bet", never "chaal" or "chal". Standard table words are fine: boot, pack, show.
- This is a scorekeeper/tracker, not a game. App name is "Teen Patti Hisaab" (Play title "Teen Patti Hisaab: Settle Up" because of the 30-char limit; App Store name "Teen Patti Hisaab" + subtitle "Score & Settle"; launcher/home-screen label "TP Hisaab"). Never use "Dealer". The upload-key folder is still named `~/TeenPattiDealer-signing` (internal only). Never describe it as a game, casino or gambling app in UI or store copy.

## Design system (tokens in `:root`)
- White page, ink `#0D1B1E`, mist `#F3F5F4` for soft fills, gold gradient for primary actions, green `#0B9467` for gains, red `#DE4247` for losses.
- Wheel looks like a casino chip: gold rim with white dashes, dark center chip showing TOTAL POT and ROUND.
- Sticky bottom dock: Undo | gold action | Reset. The gold action reads "Collect Boot" while the pot is empty and "Declare Winner" once money is in.
- Tabs under the player cards: Live Round | Scoreboard | History.

## Data model (localStorage key `teenpatti-tracker-v1`, sound pref in `teenpatti-sound`)
```
S = {
  v: 3,
  players: [{id:'p1', name:'You', color:'#E8B93A'}, ...],   // 2 to 8 players, ids are stable forever
  nextId: 4,
  boot: 10,                // starting boot, ₹1 to ₹1,00,000; v2 saves have none and migrate to 10
  potLimit: 0,             // 0 = no limit; otherwise alert + open Declare when the pot reaches it
  bet: 10,                 // current bet; never below boot; resets to boot each round
  round: 1,                // current round number
  contrib: {p1: 30, ...},  // this round's money per player
  packed: {p2: true},      // this round's packed players
  log: [{t:'bet',id,amt} | {t:'boot',ids,amt} | {t:'pack',id} | {t:'unpack',id}],  // for Undo, cleared each round
  history: [{
    round, pot, at,
    winners: ['p3'], shares: {p3: 120},   // multiple winners = split pot
    winnerId, winnerName,                  // kept for backward compatibility
    entries: [{id, name, color, paid, net, packed}]
  }],
  tab: 'live' | 'score' | 'history',
  settled: {'p2>p3:440': true}            // payments ticked as paid on the settlement screen
}
```
- `load()` migrates old saves (history with only `winnerId`, log entries without `t`, no `boot`/`potLimit`). Keep migrations working: friends already have saved games. Boot and pot limit survive "Start New Game".
- Renaming a player never breaks history because everything is keyed by id. Removed players still appear in the scoreboard/settlement with a "LEFT" chip.

## Features (all working)
- Collect Boot (gold dock button on an empty round): adds the boot for every player who isn't packed, as one undoable step.
- Tap a wheel slice or a player card to add the current bet. Chip flies into the pot, pot counts up.
- ½ Half (never below the boot) and 2× Double for the bet.
- Tap the current bet to open Table settings: starting boot (presets + typed amount) and optional pot limit (presets at 128×, 256×, 512×, 1024× the boot). Changing the boot mid-round keeps existing bets.
- Pot limit reached: brass alert, toast, then Declare Winner opens.
- Screen wake lock while the app is open (re-requested when it becomes visible again).
- Pack / Back in per player. Packed players can't bet or win. If only one player is left in, the app opens Declare Winner with them preselected.
- Undo reverses the last bet, pack or unpack.
- Reset Round (asks first if money is on the table). Reset Entire Game & History (asks first, keeps players).
- Declare Winner: pick one or several winners. Split = equal shares; leftover rupees go to the first winners in seat order.
- Reveal: drumroll overlay, then win card with crown, count-up amount, confetti (canvas physics with ₹ coins) and fanfare.
- Reopen the latest round (only when the current round is empty) to fix a wrong winner.
- Scoreboard: Won (pot shares), Lost (all money paid in), Net. Stats: rounds, total played, top pot.
- End Game & Settle Up: fewest payments between debtors and creditors (greedy), table champion, standings with bars, tap to mark paid, Copy and WhatsApp share (`https://wa.me/?text=`).
- Hand Rankings sheet (Trail > Pure Sequence > Sequence > Color > Pair > High Card, with odds).
- Edit Players: rename, recolor (picking a used color swaps), add up to 8, remove (blocked if they have bets this round).
- Pot milestone alerts at ₹500, ₹1,000, ₹2,000, ₹5,000 and up.

## Code map (inside the single `<script>`)
- State: `freshState`, `normRound`, `load`, `save`
- Sound engine: `ctx`, `osc`, `nz`, `bell`, `brass`, and `sfx.*` (clack, chip, land, boot, limit, stackUp/Down, kaching, coins, drumroll, fanfare, pack, unpack, undo, swoosh, err, flip, shuffle, hype, chime, settle). Uses Web Audio with a generated reverb. `clack` is a modal clay-chip hit (click + ceramic ring + body modes). `navigator.audioSession.type='playback'` so iOS plays sound in silent mode (note: this pauses other music apps). `ctx()` resumes any non-running state because iOS reports "interrupted" after calls/locks.
- Wake lock: `keepAwake`
- Confetti: `burst`, `fxTick`, `celebrate` on the `#fx` canvas.
- Wheel: `renderWheel`, `flyChip`, `animatePot`
- Render: `renderAll`, `renderPlayers`, `renderTabs`, `liveHTML`, `scoreHTML`, `historyHTML`
- Actions: `addBet`, `collectBoot`, `potAlerts`, `togglePack`, `undo`, `setBet`, `doResetRound`, `openDeclare`, `award`, `showWin`, `reopenRound`
- Settlement: `aggregate`, `transfers`, `shareText`, `openSettle`, `newGame`
- Sheets: `openSheet`, `closeSheet`, `confirmSheet`, `openBoot` (Table settings), `openEdit`, `openRanks`

## Store builds (`native/`, not in the public repo)
- Capacitor 8 project, appId `com.kunwar1417.teenpatti` (permanent). `npm run sync` copies `../index.html` into `native/www/`, bundles the Google Fonts locally (offline, no network) and injects `capacitor.js`.
- `index.html` has a `NATIVE` layer (null in browsers): Haptics for `buzz()`, KeepAwake, Android back button (closes sheet/win/settle, else minimizes), external links via top-level navigation.
- Build: `cd native && npm run apk` (test APK) / `npm run aab` (signed Play bundle). Needs `JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`; SDK at `/opt/homebrew/share/android-commandlinetools`.
- Upload key: `~/TeenPattiDealer-signing/` (outside the project, never commit). Bump `versionCode` in `native/android/app/build.gradle` for every Play upload.
- Icons: `native/assets/icon-src.html` renders all icon PNGs; `npm run icons` regenerates Android sizes.
- Privacy policy: `privacy.html` (linked from the app footer and Play Console).

## Ideas not built yet
- Blind / Seen mode (seen players bet 2× the current bet automatically)
- Saved game nights by date + all-time leaderboard
- Live sync between phone and tablet (would need a small free backend)
- Player stats: win rate, biggest pot, streaks
- PWA manifest + service worker so it installs and works offline

## Before you ship any change
1. Open `index.html` in a browser at phone width and play a few rounds.
2. Check: collect boot, bets, custom boot, pot limit, pack/unpack, undo, split pot, reopen round, settlement, and that every round's nets add to ₹0.
3. Make sure an old save still loads (don't rename the storage key).
