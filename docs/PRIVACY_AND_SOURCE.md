# Privacy and Source

## What this repository is

A portfolio showcase for a production application. It describes the engineering of World Clock TV on Android TV; it is not a source distribution.

## What is published

- Written descriptions of architecture, design decisions and trade-offs
- An original diagram created for this showcase
- The app icon, resized
- One design asset from the app, labelled as artwork: the TV banner illustration, which carries a fixed example time

## What is intentionally not published

- Production source code, in whole or in part
- The city dataset and any list of cities
- Build and signing configuration, certificates and keystores
- Firebase and Google service configuration, API keys and tokens
- Ad network identifiers and placement IDs
- Billing product IDs and prices
- User data of any kind
- Version control history of the production repository

## Images

- **There are no screenshots in this repository.** No captures of the app running on an Android TV or an emulator were available, and none have been mocked up or composed.
- **Everything shown is artwork.** The two images are the icon and the banner illustration, and each is captioned as artwork. The time in the banner is a fixed example, not a live reading.
- **Not republished.** Country flag images, placeholder city backgrounds and other bundled graphics were left out. The city backgrounds in the project are placeholders, and city photos are fetched from a web service at runtime in code that appears unreachable, so there is nothing genuine to show for them.
- Images were re-encoded from raw pixels so no embedded metadata remains.
- **Diagram** is an original vector illustration.

## How the descriptions were produced

The production project was inspected read-only. Statements describe behaviour observed in code, manifests and build files. Where behaviour could not be confirmed it is worded conservatively or left out. No app was run to collect measurements, and the time zone behaviour was read from the pinned package source, not exercised across a daylight saving transition.

## Publication checks

Before publishing, every file was checked for source code, credentials, signing material, personal data, email addresses, local file-system paths and embedded image metadata. None remain.

## Access

Source access for hiring or due-diligence purposes can be discussed at the owner's discretion.
