# Balule Bush Board: handover

**Written:** 1 October 2026
**Status:** for review by Glynn, who wants to build his own version

## What it is

A live trip board for the Balule trip of 27 September 2026. One shared page holds the shopping list (by store), the list of what we already have at home, a packing list, the money (who paid for what, and who owes whom) and the trip details. Everyone opens the same link, picks their name once, and sees each other's ticks and payments as they happen.

- Live page: https://robruss055.github.io/balule-bush-board/
- Usage guide for the travellers: `guide.html`, one page, written for people who have never seen the board.

## How it is built

There is no build step and no framework. The whole board is one file, `index.html`, about 60 kB, plus `guide.html`. It loads the Firebase web SDK (version 10.14.1, the `compat` builds of app, auth and firestore) from Google's CDN and talks straight to Firestore.

| Part | What it uses |
|---|---|
| Hosting | GitHub Pages, branch `main`, root folder |
| Data | Cloud Firestore, one document per trip at `trips/{TRIP_KEY}` |
| Sign-in | Firebase Anonymous Authentication, with no screen at all |
| Who you are | Chosen once from a list of names and kept in the browser's `localStorage` |
| Fonts | Google Fonts, Big Shoulders Display and Figtree |

### Storage adapter

All reads and writes go through five calls: `onCol`, `onDoc`, `update`, `add` and `del`. There are two implementations of them in `index.html`:

- `firebaseStore(cfg)` talks to Firestore. It is used when the `FIREBASE` constant at the top of the script holds a web config.
- `claudeStore(db)` talks to the database a Claude artifact is given. It is used when `FIREBASE` is empty and the page is running inside Claude.

The board was first built and tried inside Claude, then moved to Firebase so that people without a Claude account could open it. Keeping the two behind the same five calls meant no screen code changed when it moved.

### Data

Everything sits under `trips/{TRIP_KEY}`. As read from the code, not an exhaustive schema:

| Collection or document | Holds |
|---|---|
| `items` | Shopping, home and pack items: name, list, category, store, price, actual price, who it is for, whether it is ticked per person or once for everyone, who paid and when |
| `costs` | Fixed trip costs such as the gate, game drives and tolls: name, amount, quantity, who paid and when |
| `expenses` | Anything added on the road: what, amount, kind (fuel, food, drinks, activity, other), who paid, who it splits between |
| `trip/main` | The list of people and the fuel plan: kilometres there, back and local, litres per 100 km and price per litre |

### The money model

Money is never stored as a total. The page works it out each time from the ticks:

- `payments()` lists every rand that has left someone's pocket: a ticked shop item with a payer, a paid cost, or an expense.
- `stillToPay()` lists what is expected but not yet paid, using the estimates, including the fuel still to buy.
- `totals()` turns those into what each person has paid, what their share is and what they are forecast to spend.
- `settleUp()` pairs the people who are owed with the people who owe, so the fewest transfers settle the trip.

Because every figure comes from the same two lists, the totals cannot drift apart from the ticks.

## Setting up your own

Do the Firebase steps in the Firebase console and check each one there, because this list comes from the code and general Firebase practice, not from a recording of how the first board was set up.

1. Copy the repository, or create a new one and copy `index.html` and `guide.html` into it.
2. In the Firebase console, create a project. The free Spark plan is enough for a board this size.
3. Add a web app to the project and copy the config it shows you.
4. Under Authentication, enable the **Anonymous** sign-in method.
5. Under Firestore Database, create a database, then set the rules to the ones below.
6. Under Authentication settings, add your GitHub Pages domain (`<your-username>.github.io`) to the authorised domains.
7. In `index.html`, replace the `FIREBASE` config, `TRIP_KEY`, the `people` list, the `START` date, the stores and the categories.
8. Load your starting data into Firestore (see the gap below).
9. In the repository settings, turn on GitHub Pages from `main`, root folder.
10. Share the link.

The rules the code expects, from its own comment, are that any signed-in user can read and write the trip:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /trips/{trip}/{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

**Gap: there is no seed script in this repository.** The starting shopping and packing lists were put into Firestore when the board was built, and nothing in `index.html` loads them. A new board starts empty. Either enter the first items through the board itself, or write a small one-off script that adds the starting documents.

## What we learned

1. **Anonymous sign-in is what makes it easy, and it is the whole of the security.** Nobody sees a login, but anyone who has the link can read and write. That suits a list shared among a few people for a week. It does not suit anything with real client or personal data. The Firebase web config is visible in the page and in this public repository by design, so the rules are the protection, not the key.
2. **`requestAnimationFrame` does not run in a background tab.** The first version scheduled each redraw with it, so a board opened in a background tab stayed blank until the tab was brought forward. The fix (commit `7897100`) is a `setTimeout` of zero in the `schedule()` function.
3. **The first open can look empty.** The list appears only after the anonymous sign-in finishes and the first snapshot arrives. On a slow connection give it a few seconds, then refresh once. The usage guide tells people this up front.
4. **Put the numbers in one place.** Deriving every figure from ticks and payers, instead of keeping running totals, meant the totals always matched what was ticked.
5. **Write the guide for the person who has never seen it.** `guide.html` has the link with a copy button, a first-time note and one card per tab. It was the last thing added.
6. **A single file is fast to ship and slow to navigate.** At about 60 kB it is still manageable. A board with more screens would want to be split.
7. **Remove the debug logging before sharing.** The history shows a temporary debug commit, then its removal (`c9033be`, `4b14349`).

## Before you copy it

- The names of the three travellers, the stores and the trip dates are written into `index.html` and `guide.html`. Replace them before you publish your own.
- Do not reuse this Firebase project for a different board. Create your own, so the two boards cannot read or overwrite each other.
