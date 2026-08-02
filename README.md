# SwiftUI Blackjack

A complete single-player Blackjack game for iPhone, built in SwiftUI with no third-party
dependencies. Casino rules, chip betting, hand splitting, and a session scoreboard —
in a clean MVVM structure that separates the game rules from the table UI.

![Swift](https://img.shields.io/badge/Swift-5.9-orange)
![iOS](https://img.shields.io/badge/iOS-17%2B-blue)
![License](https://img.shields.io/badge/License-MIT-green)

| Place your bet | Player turn | Split hands | Round settled |
|:---:|:---:|:---:|:---:|
| <img src="Screenshots/01-betting.png" width="200"> | <img src="Screenshots/02-player-turn.png" width="200"> | <img src="Screenshots/03-split.png" width="200"> | <img src="Screenshots/04-round-over.png" width="200"> |

---

## Features

**Card game**
- Standard 52-card deck, cryptographically seeded shuffle
- Drawn cards leave the deck — duplicates are impossible
- Automatic reshuffle when the shoe runs low, excluding cards still on the table
- Aces count as 1 or 11, always resolved to the best total that doesn't bust
- Dealer AI follows house rules: hits until 17 or higher
- Hidden hole card — the dealer's score shows only the face-up total (`7 + ?`) until the reveal

**Money**
- $1,000 starting bankroll
- Chip-rack betting: $10 / $20 / $50 / $100 / $200 / $500, with chips dimming when unaffordable
- 1:1 on a regular win, **3:2 on a natural blackjack**, stake returned on a push
- Rebuy when the bankroll can't cover the minimum bet

**Split**
- Offered when the opening two cards share a rank and the bankroll covers a second bet
- Each hand carries its own bet; hand 1 is played to completion, then hand 2 activates
- Both hands render side by side, the active one boxed in gold

**Scoreboard**
- Bankroll, rounds played, and net session profit/loss in a bar above the table
- Net is colored green/red and animates with a rolling-digit transition

**Presentation**
- Dark green felt table with gold accents
- Cards slide in from the shoe and the hole card does a real 3D flip
- Every state change is animated; input is blocked while cards are moving

---

## Requirements

- Xcode 15 or later
- iOS 17+ (simulator or device)

## Run it in the simulator

No configuration, no signing setup:

```bash
open Blackjack.swiftpm
```

Pick any iPhone simulator in the toolbar and press <kbd>⌘</kbd><kbd>R</kbd>.

## Run it on a real iPhone

Deploying to a physical device needs an Apple development team ID. That ID is personal,
so this repo ships without one — `teamIdentifier` in `Package.swift` is deliberately empty.

Select your iPhone as the run destination, press <kbd>⌘</kbd><kbd>R</kbd>, and when Xcode
reports *"Signing for 'Blackjack' requires a development team"*, pick your team in the
signing editor it points you to. Xcode stores that choice in the git-ignored `.swiftpm/`
folder, so it never lands in a commit.

Alternatively, set it directly in `Package.swift`:

```swift
bundleIdentifier: "com.yourname.blackjack",
teamIdentifier: "YOURTEAMID",
```

Your team ID is the 10-character code in **Xcode ▸ Settings ▸ Accounts ▸ Manage
Certificates**, or in the `OU` field of:

```bash
security find-identity -v -p codesigning
```

If you edit `Package.swift` this way, leave the change uncommitted (`git update-index
--skip-worktree Blackjack.swiftpm/Package.swift` keeps it out of `git status`).

A free Apple ID works. Its provisioning profiles expire after 7 days, after which the app
stops opening on the phone until you run it from Xcode again.

---

## Project structure

```
Blackjack.swiftpm/
├── Package.swift               App manifest + local signing lookup
├── BlackjackApp.swift          @main entry point
├── Models/
│   ├── Card.swift              Suit, Rank, Card
│   ├── Hand.swift              handValue(), HandOutcome, PlayerHand
│   └── Deck.swift              Shuffling, drawing, auto-reshuffle
├── ViewModel/
│   └── BlackjackViewModel.swift  Game state machine, betting, split, dealer AI, payouts
└── Views/
    ├── Theme.swift             Every color and metric in one place
    ├── ContentView.swift       Table layout
    ├── CardView.swift          Card face, patterned back, 3D flip
    ├── ChipView.swift          Casino chip
    ├── ScoreboardView.swift    Bankroll · rounds · net
    ├── HandView.swift          Card rows, hand headers, split hands
    ├── BettingView.swift       Bet display + chip rack
    └── ControlsView.swift      Phase-driven button row
```

The game rules live entirely in `BlackjackViewModel` — the views read published state and
call intent methods (`addChip`, `deal`, `hit`, `stand`, `split`, `nextRound`). Every view
file ships with an Xcode `#Preview`, so pieces can be designed in isolation.

---

## How the core logic works

### Ace valuation

Every ace starts at 11, then demotes to 1 (subtract 10) one at a time while the hand is
over 21. This always produces the highest legal total:

```swift
func handValue(of hand: [Card]) -> Int {
    var total = hand.reduce(0) { $0 + $1.rank.value }
    var flexibleAces = hand.filter { $0.rank == .ace }.count
    while total > 21 && flexibleAces > 0 {
        total -= 10
        flexibleAces -= 1
    }
    return total
}
```

### No duplicate cards

`Deck.draw` removes the card it returns. When the shoe drops to 10 cards it rebuilds a
fresh 52-card deck and removes one copy of every card currently on the table before
shuffling — so a mid-round reshuffle can never deal a second copy of a card you're holding.

### Round state machine

```
betting ──deal()──▶ dealing ──▶ playerTurn ──stand/bust──▶ dealerTurn
   ▲                   │            │  ▲                        │
   │                   │ natural    │  │ split() / next          │
   │                   ▼ blackjack  ▼  │ split hand              ▼
   └──nextRound()── roundOver ◀──────┴──────────────── settleRound()
```

`.dealing` doubles as a busy state: whenever cards are animating, taps are ignored, which
keeps double-taps from desynchronizing a split.

### Payouts

The bet leaves the bankroll at **Deal** (and again at **Split**); money only returns
through `settleRound()`:

| Result | Returned |
|---|---|
| Natural blackjack (un-split, first two cards) | stake + 3:2 |
| Win | stake + 1:1 |
| Push | stake |
| Lose / bust | nothing |

Only an un-split opening 21 counts as a natural, matching casino rules — a 21 made from
split cards pays even money.

---

## License

MIT — see [LICENSE](LICENSE).
