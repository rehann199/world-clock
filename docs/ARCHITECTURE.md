# Architecture

Conceptual view of the Android TV app. Names are descriptive, not production identifiers.

![System overview](../assets/diagrams/system-overview.svg)

## Layers

| Layer | Responsibility |
|---|---|
| Input | Per-screen key handlers translate D-pad, select and back into grid moves, focus changes or navigation |
| Screens | Home grid, city detail, manage cities, add city, settings (plus a few registered routes that are not reached) |
| Clock state | One notifier holding a map of city clocks, updated by a one-second timer, plus a stream for the header's local time |
| Time zone data | The `timezone` package's all-zones database, one `Location` per zone identifier, cached |
| Data | Bundled SQLite city database (read-only copy in app storage), CSV fallback, key-value preferences |
| Services | Weather and photo requests, IP-based first-city lookup, connectivity check, ads, analytics, crash reporting, billing client in Dart |
| Native | Kotlin billing manager, keep-awake handling, screensaver service, activity host |

## Startup

```mermaid
sequenceDiagram
    participant App as main()
    participant FB as Firebase
    participant Env as Config load
    participant Ads as Ads (background)
    participant St as App state
    participant DB as City DB
    participant Net as IP location
    App->>FB: init, enable analytics and crash collection
    App->>App: immersive system UI
    App->>Env: load bundled config
    App->>Ads: init and preload (not awaited)
    App->>App: open preferences, read package info
    App->>St: run app, build state
    St->>St: init billing, refresh entitlements (failures logged, not fatal)
    St->>DB: first run only, nearest-city query
    St->>Net: first run only, approximate location (timeout)
    Net-->>St: position, or failure and default city
```

## The clock path

```mermaid
flowchart LR
    I[City record<br/>IANA id] --> L[Location lookup<br/>cached per id]
    L --> N[TZDateTime.now<br/>first tick]
    N --> M[(Clock map)]
    T[1 s timer] --> A[Add one second<br/>to cached value]
    A --> M
    M --> H[Home cards<br/>12-hour text]
    M --> D[City detail]
    H -. label .-> U[Stored UTC string<br/>set when city was added]
```

The dotted edge matters: the label on screen is read from the stored city record, not from the clock map.

## Search path

```mermaid
flowchart TD
    Q[Typed query] --> D[Debounce 220 ms]
    D --> Nm[Normalise<br/>fold accents, strip punctuation]
    Nm --> R{SQLite query}
    R -->|ok| C[De-duplicate, cap 30, cache]
    R -->|error| F1[Parse bundled CSV<br/>rank in Dart]
    F1 -->|empty or error| F2[Small seed list]
    C --> UI[Result list with focus nodes]
    F1 --> UI
    F2 --> UI
```

## Focus model on the home screen

```mermaid
flowchart TD
    H[Header<br/>manage, settings] -->|Down| G[Grid, first row of page]
    G -->|Up from first row, first page| H
    G -->|Up from first row, later page| P[Previous page, last row]
    G -->|Down from last row| NP[Next page, same column]
    G -->|Select| X{Slot kind}
    X -->|city| Dt[City detail]
    X -->|empty| Mg[Manage cities]
    X -->|locked| Pu[Purchase dialog]
```

## Native bridge

Three method or event channels sit on the activity: one for billing calls, one for billing events (ready, disconnected, products, purchase status, success, error, cancel) and one for screensaver queries. The billing manager reconnects after a disconnect, queries product details and purchases, acknowledges purchases and caches a premium flag locally. A legacy billing channel name is also still registered.

## Boundaries and failure modes

- Ads, weather and the first-launch location lookup are best effort. Each has a timeout or a fallback, so a failure leaves the clock grid usable. On a first run the app state is still loading while the location lookup runs (up to eight seconds), so the first city can appear late; this was read from the code, not timed.
- Billing initialisation failure is caught and logged, and the app continues with locally stored entitlements.
- Search degrades from SQLite to CSV to a seed list.
- Analytics calls are wrapped so a logging failure cannot reach the UI.
