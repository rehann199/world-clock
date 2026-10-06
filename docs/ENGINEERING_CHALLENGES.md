# Engineering Challenges

Six problems that stand out in the Android TV app. Each is drawn from behaviour visible in the code. Outcomes describe what the code does; no metrics are claimed, and the app was not run to measure anything.

---

## 1. Correct wall time for every city, in 400 zones

**Problem**
A world clock has to show the right local time for places with half-hour and 45-minute offsets and with daylight saving rules that differ by region and change over the years. A table of fixed offsets would be wrong for part of the year in many places.

**Context**
The city data is a bundled table of about 48,000 places. Every row has an IANA time zone identifier, and the data contains 400 distinct identifiers.

**Approach**
- Use the `timezone` package's all-zones database instead of an offset table. Initialise it once, look up a `Location` per identifier, and cache the lookup.
- Build a `TZDateTime` from the current instant in the city's location, and format the clock text from that value.
- If an identifier cannot be resolved, fall back to UTC rather than throw.
- Compute the "UTC +HH:MM" text with `timeZoneOffset` at the moment a city is added.

**Trade-offs**
The offset text is computed once and stored with the city, so it can show the wrong season's offset after a DST change. The live clock map carries an offset string too, but the cards read the stored one. The rule data is fixed at the version packaged with the app. Nothing tests any of this.

**Outcome**
The displayed time is rule-driven for every one of the 400 zone identifiers in the data. The label is the weak point and the fix is small: read the offset from the live value.

---

## 2. A one-second clock on many cards

**Problem**
Up to nine cards tick every second on TV hardware, and the cost of resolving a time zone for every card every second is unnecessary.

**Context**
The app uses Riverpod. Home needs a map of clocks for the visible cities and a separate local-time value for its header.

**Approach**
- One async notifier owns a single periodic timer and a cache keyed by city id.
- The first refresh per city resolves the zone and converts the instant. Later ticks add one second to the cached value, which the package re-resolves through the location when it builds the new value.
- The timer is cancelled when the notifier is disposed.

**Trade-offs**
The home screen reads both the clock map and the local-time stream in its build method, so the whole home screen rebuilds on each tick. No repaint boundaries were found. The cache is not re-anchored to the system clock, so delayed or skipped ticks would not be corrected in the code that was read.

**Outcome**
One timer drives all cards and zone lookups are not repeated per tick. The rebuild scope and the missing resync are the two places to improve; neither was profiled or measured.

---

## 3. A paged grid, a header, dialogs and a text field on one remote

**Problem**
A remote cannot tap. The home screen has a header row with two icons, a 3 by 3 grid that pages, locked and empty slots, and dialogs. The add-city screen needs typing on an on-screen keyboard and then moving back to the results with arrows.

**Context**
Flutter's geometric focus traversal is hard to predict with pages, overlapping scale animations and a text field that captures arrow keys.

**Approach**
- Home keeps a slot index and a header-focus flag and handles every arrow itself, including page changes.
- The add-city screen uses named focus nodes for the back button, search bar, text input and each result. It opens the keyboard explicitly, closes it on Up, Down, Back or submit, and makes the text input non-focusable while closed so arrows are not swallowed.
- Dialogs request focus on their main button after the first frame and dispose their nodes on close.
- A shared focusable wrapper scales and outlines the focused child and reports key actions.

**Trade-offs**
Movement rules are code, so a new control means editing the handler. Home is a 770-line file and there are no tests for movement. Touch handlers coexist with the key handlers.

**Outcome**
Movement across rows, pages, header and dialogs is explicit and deterministic, and the search flow returns focus to a known widget after the keyboard closes.

---

## 4. Searching a large place list quickly

**Problem**
About 48,000 places have to be searchable by a query typed slowly with a remote, with accents and punctuation ignored, and with sensible ordering.

**Context**
Parsing a 5 to 6 MB CSV on each query would be wasteful. The data also has duplicates by name, country and region.

**Approach**
- A build tool converts the CSV into a SQLite database with normalised columns and indexes, and the app ships it as an asset.
- The app copies it to its database directory on first use and opens it read-only.
- Queries are debounced, folded to ASCII lower case, capped, ranked (exact, prefix, substring, then capitals, then population), de-duplicated and cached for recent queries.
- If SQLite fails, parse the CSV in Dart with the same ranking, then fall back to a small seed list.

**Trade-offs**
Substring matches cannot use ordinary indexes. Both the database and the CSV ship, which adds size. Three code paths implement ranking, and only the SQL and Dart variants are alike. No tests cover search.

**Outcome**
Search has a ranked order and two fallbacks, and no error reaches the UI without a degraded result path.

---

## 5. Keeping ads and network calls from blocking a TV

**Problem**
A full-screen ad that never calls back, or a weather request that hangs, would leave a living-room TV stuck.

**Context**
The app shows interstitials on Home on a timer and offers a rewarded path, and it requests weather per visible city.

**Approach**
- Ads initialise in the background with a 12-second timeout. Loads retry with exponential backoff capped at 30 seconds. A show waits at most two seconds for a load, retries once and then calls the completion callback regardless.
- Interstitials are skipped when the user has subscribed or removed ads, and a cooldown applies.
- Weather failures return a null that the card shows as a dash.
- HTTP calls have explicit connect and receive timeouts.

**Trade-offs**
Ads initialise at startup even for users who have removed ads, and Home's timer fires every 15 minutes while ads are enabled. Weather is never refreshed during a session.

**Outcome**
Ad and network failures are fail-open: the clock grid stays usable.

---

## 6. Keeping the screen on for a clock

**Problem**
A clock is only useful if the TV does not sleep or start its own screensaver while the app is in front.

**Context**
Some TV builds ignore the standard keep-screen-on flag.

**Approach**
- Set the window flag and the decor view flag on resume.
- Also hold a wake lock with a ten-hour timeout, and release everything on pause and destroy.
- Declare a screensaver service that launches the main activity.

**Trade-offs**
Keeping the screen on with static content raises burn-in risk, and no mitigation (dimming, pixel shifting) was found in the code, so none is claimed. The wake lock uses a legacy type. The native screensaver channel has no Dart caller in the code read.

**Outcome**
Three layers keep the display awake while foregrounded and are released when the app pauses.
