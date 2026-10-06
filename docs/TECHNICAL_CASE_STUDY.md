# Technical Case Study

World Clock TV on Android TV. Everything below is drawn from reading the production code, manifest and build files. The app was not run to collect measurements, and no timings or user numbers are claimed. Where a point could not be confirmed it is qualified or left out.

No Android TV screenshots are included in this repository; images are artwork (see [Privacy and source](PRIVACY_AND_SOURCE.md)).

## 1. Scope

A single-activity Flutter app for TV with a small Kotlin layer. About 8,100 lines of Dart across 36 files and about 780 lines of Kotlin across 4 files. 5 unit tests in 2 files cover the analytics sanitiser and the unlock helpers. There are no tests for clocks, time zones, search or focus movement.

## 2. Time zones: where the correctness comes from

### What the code uses

| Question | Answer from the code |
|---|---|
| Source of zone rules | The `timezone` Dart package, loaded with its all-zones data file (the package describes it as built from the IANA time zone database) |
| Hand-written offset tables | A small seed list of nine cities has hand-typed offset strings, but that list is not wired into any provider at runtime (the city list provider returns an empty list), so it is dead code for display purposes |
| Platform time zone APIs | Not used for city clocks. The device's own local time (`DateTime.now`) drives only the home header clock |
| Zone identifier per city | A column in the bundled city data. The data has 400 distinct identifiers across about 48,000 rows |
| Failure behaviour | If a zone id cannot be resolved, the lookup falls back to UTC for the clock, and to `+00:00` for the label |

### Where the conversion happens

1. At startup of the clock controller the time zone database is initialised once (a static flag guards repeated loading).
2. For each selected city, a `Location` is looked up by identifier and cached in a map.
3. On the first refresh for a city, `TZDateTime.now(location)` converts the current instant into that city's wall time. That value and the formatted offset string are cached.
4. A periodic one-second timer then runs. On each tick, for a city already in the cache, the code adds one second to the cached `TZDateTime` and keeps the stored offset string. For a city not yet in the cache (for example one just enabled) it resolves the zone afresh.
5. The card formats the cached value with a 12-hour pattern (`hh:mm:ss a`, English).

### Daylight saving time: what is and is not verified

- The wall time shown is derived from `TZDateTime`, which applies the rule data for that location. In the version of the package that the project pins, `add` converts the underlying instant and re-applies the location, so adding a second near a transition resolves against the rules rather than against a fixed offset. This was established by reading the package source, not by running the app across a transition.
- The UTC label on each card (`UTC +HH:MM`) is not taken from the live clock value. It is the `utcOffset` string stored on the city record, computed with `TZDateTime.now(location).timeZoneOffset` when the city was added and then saved with the city in preferences. A city added in one season therefore keeps that season's label. The live clock map also carries an offset string, but nothing on screen reads it.
- Rule data is the version bundled with the pinned package, so a rule change after release reaches users only with an app update.
- There is no automated test for any of the above.

### Drift and resynchronisation

The one-second increment is applied to the cached value, and the code read does not re-anchor the cache to the system clock after the first tick for a city. Timers in Dart do not promise exact one-second spacing, and the code does not handle the case of ticks being delayed while the app is paused. The consequence of this was not measured. A resynchronisation from the system clock on resume or on each tick would remove the question.

### Formatting and localisation

- Time uses `hh:mm:ss a` and dates use English patterns with the locale fixed to `en_US` in the shared formatter. The home header formats with the default locale.
- There is no 24-hour setting and no localisation resources were found, so the interface strings are hard-coded English.

## 3. City data and search

- The bundled data has about 48,000 places across 237 country codes. A build tool converts a CSV into a SQLite database with normalised name columns and indexes; both the database and the CSV ship as assets.
- On first use the database is copied from assets to the app's database directory and opened read-only.
- A query is trimmed, folded to ASCII lower case, stripped of punctuation, debounced by 220 ms, and run as a contains match. Results sort exact match first, then prefix, then substring, then capitals before others, then by population. The repository caps results at 30 after de-duplicating by name, country and region, keeps a cache of recent queries (bounded at 80) and excludes cities already managed.
- Contains matches (`%query%`) cannot use an ordinary index, so the indexes mainly help ordering and exact and prefix cases. No timing was measured.
- If the database call throws, the controller falls back to parsing the CSV asset (cached in memory) and ranks in Dart, then to the seed list.
- The first launch seeds a city: it asks a public IP-location service for an approximate position (6-second timeouts, 8 seconds overall), takes the nearest place by squared coordinate difference, and falls back to New York, London or Tokyo, then to the largest capital in the data.

## 4. Favourites and multiple clocks

- Two ordered lists are stored in preferences: managed cities (what the user added) and selected cities (what Home shows). Selection is capped at 25 and the cap raises a snack bar message.
- Custom cities are serialised to a JSON map in preferences, including the stored offset string.
- Startup runs a reconciliation step that drops ids that no longer resolve and migrates older id formats.
- The home grid has 25 slots; locked slots show a lock and open a purchase dialog.

## 5. The live clock and rebuild scope

- One `Timer.periodic` (1 s) inside an async notifier updates a map of city clocks. A separate `Stream.periodic` provider supplies the header's local time.
- The home screen reads both providers in its `build`, so the whole home screen, including all visible cards, is rebuilt at least once per second and probably twice, because the two providers tick independently. Cards are stateless and receive new data. No repaint boundaries were found.
- Each visible card also watches a per-city weather future. Those providers are not auto-disposed and are never invalidated in the code read, so weather is fetched once per city per session and not refreshed.
- Rebuilding a 3 by 3 page every second is probably adequate on typical TV hardware, but this was not profiled.

## 6. TV input and focus

- Home: one `Focus` with a key handler. State is a slot index and a header-focus flag. Right and Left move within a row, Down and Up move by a row and across pages, and the header has two icons with Left and Right. Select opens a city, a manage screen, settings, a purchase dialog or the add flow depending on what is focused.
- Focused cards get a border and shadow; the focused element is the one the code says it is, not what geometric traversal would choose.
- A shared wrapper (`TvFocusable`) scales to 1.05, outlines the focused child, reports arrow, back and select presses to analytics, and handles the select key. Other keys are ignored so the framework can move focus.
- Add city: named focus nodes for the back control, search bar, text input and each result. Entering the field opens the on-screen keyboard explicitly; Down, Up or Back close it and move focus to the results or back button; the input is made non-focusable while the keyboard is closed so arrow keys do not get stuck in it.
- Dialogs: the purchase dialog requests focus on its first button after the first frame and disposes its node on close. The privacy dialog handles Up and Down scrolling and moves focus to its Close button at the end.
- Touch handlers (`GestureDetector`) are present on cards and icons, but the key paths are primary.
- Back uses the framework default for routes; no custom back interception was found on Home.

## 7. Lifecycle, keep-awake and the screen

- The native activity sets the keep-screen-on window flag on resume, sets it on the decor view, and holds a screen-bright wake lock (ten-hour timeout, not reference counted), releasing all three on pause. The wake lock is released again on destroy.
- A screensaver (Dream) service is declared in the manifest. When started it launches the main activity with a flag and finishes. The native side has a channel to check, set and open screensaver settings, but no Dart caller was found and the extra flag is not read by the code.
- No dimming, pixel shifting, inverted theme at idle or other burn-in mitigation was found. The app keeps the screen on while foregrounded, which is the behaviour of a clock display, and does not do anything to limit static content.
- Dart timers are not stopped on pause by the app's own code; behaviour while backgrounded depends on the platform.

## 8. Ads

- Unity Ads is initialised from the bootstrap sequence regardless of the entitlement state, and again conditionally from the app state controller. Home shows an interstitial on a 15-minute periodic timer only when ads are enabled and the user has neither subscribed nor removed ads.
- The ads layer is fail-open: init has a 12-second timeout, loads retry with exponential backoff from 2 to 30 seconds, a show waits up to 2 seconds for a load, retries once and then calls the completion callback anyway. A 30-second cooldown applies between interstitials. Failures are logged to analytics.
- The ads toggle exists in the settings model (default on) but no control for it was found in the settings screen.

## 9. Analytics and diagnostics

- Firebase Analytics and Crashlytics collection are enabled unconditionally at startup. No consent step was found in the code.
- A sanitising layer validates event and parameter names, drops reserved names, de-duplicates repeated events within a short window, and measures screen durations. Tests cover the sanitiser and the de-duplication.
- Framework and platform errors are forwarded to Crashlytics as fatal.
- Events that include the selected city's id, name, zone id and offset are logged for city actions.

## 10. Native code

About 780 lines of Kotlin: a billing manager (reconnect, product queries, purchase flow, acknowledgement, cached ownership), the activity that wires three channels (billing methods, billing events, screensaver) and handles keep-awake, a trivial application class and the screensaver service.

## 11. Release configuration

Release builds use minification and resource shrinking with a keep-rules file for the Flutter and app entry points. Signing is read from a local properties file that is not committed. Compile and target SDK follow the Flutter toolchain, minimum SDK is 24 and Java 17 is used.

## 12. Things worth fixing

Stated plainly, in order of how much they matter to correctness or maintenance:

1. Show the live offset on cards, or recompute the stored label, so it cannot go stale across a DST change.
2. Re-anchor the cached clocks to the system clock instead of incrementing indefinitely.
3. Split the home build so only the clock text rebuilds each second.
4. Add tests for zone conversion, formatting and focus movement.
5. Add a 24-hour option and externalise strings.
6. Remove or wire up the dead seed list and the unreachable routes.
