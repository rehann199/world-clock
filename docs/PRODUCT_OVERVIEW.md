# Product Overview

World Clock TV is a world clock for Android TV. The home screen is a grid of city cards, each with a live time, and every control is reachable with the remote's D-pad and select button.

No Android TV screenshots are included in this repository; the images used are artwork (see [Privacy and source](PRIVACY_AND_SOURCE.md)).

## Platform

- Android TV only. The manifest requires the Leanback feature, declares that a touchscreen is not required, registers a Leanback launcher activity and provides a TV banner.
- Landscape orientation is fixed in the manifest. Minimum SDK is 24.
- The system bars are hidden (immersive mode) at startup.
- Flutter and Dart for the UI, with Kotlin for billing, keep-awake handling and a screensaver service.

## Screens

| Screen | What it does |
|---|---|
| Home | A 3 by 3 page of city cards with a header showing the local time and date, a manage button and a settings button. Up to 25 slots, paged by nine |
| City detail | Back control, city and country, long date, time with seconds, UTC label, weather metric cards and country facts (capital, population, currency) |
| Manage cities | The list of managed cities with a toggle for which are shown on Home, an add button, and a limit of 25 enabled |
| Add city | A search field and a ranked list of results with flags |
| Settings | Slot bundles, remove ads, dark mode, a link to the terms, the privacy policy as an in-app dialog, and a list of the developer's other apps that opens store links |

## Time and data

- Each place carries an IANA time zone identifier. The bundled data has about 48,000 places, 237 country codes and 400 distinct time zone identifiers (counted from the bundled files).
- The card time is formatted as `hh:mm:ss a` with English month and day names. No 24-hour setting and no translated interface were found.
- Weather comes from a third-party weather web API and is best effort: if the request fails the card shows a dash. Country facts on the detail screen come from the bundled data.
- On the first run the app calls a public IP-location service, finds the nearest place in the bundled database and adds it as the first city. If that fails it picks a default from a short list of well-known cities.

## Slots and monetisation

- The first two home slots are free. Further slots unlock through one-time per-slot products, or three bundles that unlock more at once.
- A separate remove-ads purchase disables interstitials on Home, and a subscription product identifier also exists in the code.
- Interstitial ads are attempted on a 15-minute timer on Home when ads are enabled and the user has not subscribed or removed ads. A rewarded-ad path and a one-hour temporary unlock exist in the entitlement model, but the screen that would use them is not reached (see the last section).
- Billing is handled by a native Kotlin manager exposed to Dart through channels, and ownership is cached locally.

## Keeping the screen on

While the main activity is in the foreground it sets the keep-screen-on flag and holds a wake lock, and releases both when it pauses. A screensaver (Dream) service is declared that launches the main activity. No pixel-shifting, dimming or other burn-in mitigation was found in the code, so none is claimed.

## What it is not

The home header clock uses the device's own local time from the system clock; the city cards use the per-city zone data. The app does not offer alarms, timers or a 24-hour mode in the code that was read.

## Code that is present but not reached

The route table also registers a photo view, an unlock screen and an other-apps screen, and a city-photo web request and a country-facts request exist as providers. In the code that was read, nothing navigates to those screens or consumes the country-facts provider, and the photo and unlock screens look cities up in a list that is empty at runtime. They are described here only so the inventory is accurate; they are not presented as features.
