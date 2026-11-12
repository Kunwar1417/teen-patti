# Teen Patti Hisaab

A single-file web app (`index.html`) for tracking bets during Teen Patti nights with friends. Used on phones and tablets. Hosted on GitHub Pages at `https://kunwar1417.github.io/teen-patti/` and packaged for Google Play from `native/` (see below). Current version: 1.1.

## Ground rules
- Keep it ONE self-contained `index.html` (inline CSS + JS). No build step, no frameworks, no npm, no backend unless explicitly asked. (The `native/` store wrapper is the only npm project and never changes how `index.html` is written.)
- Only external loads: Google Fonts (Bricolage Grotesque for display, Manrope for body, Baloo 2 for Devanagari only). Everything else (sounds, icons, confetti) is generated in code.
- Mobile first: test at iPhone width (390px). No horizontal page scroll; only tables may scroll sideways inside their own container.
- Money is whole rupees (integers). Every round's nets must sum to exactly ₹0. Never break this.
- Never use em dashes in any user-facing text (either language).
- Plain, friendly English in the UI. Say "bet", never "chaal" or "chal". Standard table words are fine: boot, pack, show, blind, seen.
- Every user-facing string goes through `t(key, vars)` with an entry in BOTH `I18N.en` and `I18N.hi`. Names put into strings that become innerHTML must be `esc()`'d by the caller. Hindi terms: bet = दांव (not चाल), boot = बूट, pack = पैक, back in = वापस, show = शो, blind = ब्लाइंड, seen = सीन, pot = पॉट, winner = विजेता. Amounts stay `en-IN` digits with ₹.
- This is a scorekeeper/tracker, not a game. App name is "Teen Patti Hisaab" (Play title "Teen Patti Hisaab: Settle Up" because of the 30-char limit; App Store name "Teen Patti Hisaab" + subtitle "Score & Settle"; launcher/home-screen label "TP Hisaab"). Developer: Never Settle Lab, neversettlelab@gmail.com. Never use "Dealer". The upload-key folder is still named `~/TeenPattiDealer-signing` (internal only). Never describe it as a game, casino or gambling app in UI or store copy.

## Design system (tokens in `:root`)
- White page, ink `#0D1B1E`, mist `#F3F5F4` for soft fills, gold gradient for primary actions, `--gold-soft` for selected/seen states, green `#0B9467` for gains, red `#DE4247` for losses.
- Wheel looks like a casino chip: gold rim with white dashes, dark center chip showing TOTAL POT and ROUND. Slice labels show PACKED or 👁 SEEN.
- Header: brand + three badge icons: Hand rankings (cards + gold bulb badge), Players (people + gold + badge), Settings (gear).
- Fixed bottom dock: on the Table tab an action row Undo | gold action | Reset (gold reads "Collect Boot" on an empty pot, "Declare Winner" once money is in), and always a bottom tab bar Table | Scores | History. `body.tab-live` adjusts page padding and toast position.
- Table tab: stake card, wheel, player cards (Pack pill; the eye toggle for blind/seen appears only when Blind mode is on), then a fold-out "Round details" (the old Live Round table; open state in `teenpatti-details`).
- Scores tab ends with a quiet "Delete all rounds and start over" link (the only place to reset the whole game besides Settle Up's "Start New Game").
- Settings sheet: rows with icon tiles (`.sico`): Language (English | हिंदी), Sound switch (speaker icon turns gold when on), Table rules row (boot, pot limit, Blind mode), then about/privacy/version.
- Table rules sheet (tap the current bet, or Settings → Table rules): starting boot, pot limit, Blind mode switch.
- Hindi: `html[lang="hi"]` removes letter-spacing from small caps labels (Devanagari must never be letter-spaced) and nudges small text up a size.

## Data model (localStorage key `teenpatti-tracker-v1`; device prefs: `teenpatti-sound`, `teenpatti-lang`, `teenpatti-details`)
```
S = {
  v: 4,
  players: [{id:'p1', name:'You', color:'#E8B93A'}, ...],   // 2 to 8 players in SEAT ORDER (drag to reorder); ids are stable forever
  nextId: 4,
  boot: 10,                // starting boot, ₹1 to ₹1,00,000; v2 saves have none and migrate to 10
  potLimit: 0,             // 0 = no limit; otherwise alert + open Declare when the pot reaches it
  blindMode: false,        // table rule, default OFF (most groups play seen). Off = no eye buttons, every tap adds the bet. Survives New Game.
  bet: 10,                 // current (blind) bet; never below boot; resets to boot each round
  round: 1,                // current round number
  contrib: {p1: 30, ...},  // this round's money per player
  packed: {p2: true},      // this round's packed players
  seen: {p3: true},        // this round's seen players: their taps add 2 × bet. Ignored unless blindMode (isSeen checks both). v3 saves have none.
  log: [{t:'bet',id,amt} | {t:'boot',ids,amt} | {t:'pack',id} | {t:'unpack',id} | {t:'seen',id} | {t:'blind',id}],  // for Undo, cleared each round
  history: [{
    round, pot, at,
    winners: ['p3'], shares: {p3: 120},   // multiple winners = split pot
    winnerId, winnerName,                  // kept for backward compatibility
    entries: [{id, name, color, paid, net, packed, seen}]
  }],
  tab: 'live' | 'score' | 'history',      // 'live' is the Table tab
  settled: {'p2>p3:440': true}            // payments ticked as paid on the settlement screen
}
```
- `load()` migrates old saves (history with only `winnerId`, log entries without `t`, no `boot`/`potLimit`, no `seen`). Keep migrations working: friends already have saved games. Boot and pot limit survive "Start New Game".
- Renaming or reordering players never breaks history because everything is keyed by id. Removed players still appear in the scoreboard/settlement with a "LEFT" chip.

## Features (all working)
- Collect Boot (gold dock button on an empty round): adds the boot for every player who isn't packed, as one undoable step.
- Tap a wheel slice or a player card to add that player's bet (the current bet; 2× for seen players in Blind mode). Chip flies into the pot, pot counts up.
- Blind mode (Table rules switch, off by default): everyone starts each round blind; the eye button on a card marks them seen and their taps become 2×. Stake card shows "Seen players pay ₹X" while anyone is seen. Undoable; cleared each round; saved in history and restored by Reopen. Turning Blind mode off clears the round's seen marks (bets stay). With it off, cards read "Tap +₹X".
- ½ Half (never below the boot) and 2× Double for the bet.
- Tap the current bet (or Settings → Table rules) to open Table rules: starting boot (presets + typed amount) and optional pot limit (presets at 128×, 256×, 512×, 1024× the boot). Changing the boot mid-round keeps existing bets.
- Pot limit reached: brass alert, toast, then Declare Winner opens.
- Screen wake lock while the app is open (re-requested when it becomes visible again).
- Pack / Back in per player. Packed players can't bet or win. If only one player is left in, the app opens Declare Winner with them preselected.
- Undo reverses the last bet, boot collection, pack, unpack, seen or blind.
- Reset Round (asks first if money is on the table). Delete all rounds from the Scores tab (asks first, keeps players).
- Declare Winner: pick one or several winners. Split = equal shares; leftover rupees go to the first winners in seat order.
- Reveal: drumroll overlay, then win card with crown, count-up amount, confetti (canvas physics with ₹ coins) and fanfare.
- Reopen the latest round (only when the current round is empty) to fix a wrong winner.
- Scores tab: Won (pot shares), Lost (all money paid in), Net. Stats: rounds, total played, top pot.
- End Game & Settle Up: fewest payments between debtors and creditors (greedy), table champion, standings with bars, tap to mark paid, Copy and WhatsApp share (`https://wa.me/?text=`) in the chosen language.
- Hand Rankings sheet (Trail > Pure Sequence > Sequence > Color > Pair > High Card, with odds), in both languages.
- Players sheet: hold the ⠿ grip and drag to change seat order (arrow keys also work), rename, recolor (picking a used color swaps), add up to 8, remove (blocked if they have bets this round).
- English / हिंदी switch in Settings; default player names follow the language at creation.
- Pot milestone alerts at ₹500, ₹1,000, ₹2,000, ₹5,000 and up.

## Code map (inside the single `<script>`)
- Language: `I18N` (en, hi), `LANG`, `t`, `applyStatic` (fills `data-i18n` / `data-i18n-aria`), `setLang`
- State: `freshState`, `normRound`, `load`, `save`; helpers `isSeen`, `betFor`
- Native layer: `NATIVE`, `buzz`, `openExternal`
- Sound engine: `ctx`, `osc`, `nz`, `bell`, `brass`, and `sfx.*` (clack, chip, land, boot, limit, stackUp/Down, kaching, coins, drumroll, fanfare, pack, unpack, undo, swoosh, err, flip, shuffle, hype, chime, settle). Uses Web Audio with a generated reverb. `clack` is a modal clay-chip hit (click + ceramic ring + body modes). `navigator.audioSession.type='playback'` so iOS plays sound in silent mode (note: this pauses other music apps). `ctx()` resumes any non-running state because iOS reports "interrupted" after calls/locks. `toggleSound` flips it from Settings.
- Wake lock: `keepAwake`
- Confetti: `burst`, `fxTick`, `celebrate` on the `#fx` canvas.
- Wheel: `renderWheel`, `flyChip`, `animatePot`
- Render: `renderAll`, `renderPlayers`, `renderTabs`, `switchTab`, `renderDetails`, `liveTable`, `scoreHTML`, `historyHTML`
- Actions: `addBet`, `collectBoot`, `potAlerts`, `togglePack`, `toggleSeen`, `undo`, `setBet`, `doResetRound`, `openDeclare`, `award`, `showWin`, `reopenRound`
- Settlement: `aggregate`, `transfers`, `shareText`, `openSettle`, `newGame`
- Sheets: `openSheet`, `closeSheet`, `confirmSheet`, `confirmResetAll`, `openBoot` (Table rules: boot, pot limit, Blind mode), `openEdit` + `bindReorder` + `movePlayer`, `openRanks`, `openSettings`

## Store builds (`native/`, not in the public repo)
- Capacitor 8 project, appId `com.kunwar1417.teenpatti` (permanent). `npm run sync` copies `../index.html` into `native/www/`, bundles the Google Fonts locally (latin, latin-ext for ₹, devanagari for Hindi; offline, no network) and injects `capacitor.js`.
- `index.html` has a `NATIVE` layer (null in browsers): Haptics for `buzz()`, KeepAwake, Android back button (closes sheet/win/settle, then goes back to the Table tab, else minimizes), external links via top-level navigation.
- Build: `cd native && npm run apk` (test APK) / `npm run aab` (signed Play bundle). Needs `JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`; SDK at `/opt/homebrew/share/android-commandlinetools`.
- Upload key: `~/TeenPattiDealer-signing/` (outside the project, never commit). Bump `versionCode` in `native/android/app/build.gradle` for every Play upload after the first; keep `versionName` and `APP_VERSION` in `index.html` in step.
- Icons: `native/assets/icon-src.html` renders all icon PNGs; `npm run icons` regenerates Android sizes.
- Store kit: `native/store/` (listing text, screenshots, feature graphic). Privacy policy: `privacy.html` (linked from Settings and Play Console).

## Ideas not built yet
- Saved game nights by date + all-time leaderboard
- Live sync between phone and tablet (would need a small free backend)
- Player stats: win rate, biggest pot, streaks
- First-game setup (type names + pick boot on first open)
- iOS App Store build from the same `native/` project (needs Xcode)

## Before you ship any change
1. Open `index.html` in a browser at phone width and play a few rounds, in English and in Hindi.
2. Check: collect boot, bets, blind/seen, custom boot, pot limit, pack/unpack, undo, split pot, reopen round, drag to reorder, settlement, tabs, and that every round's nets add to ₹0.
3. Make sure an old save still loads (don't rename the storage key).
