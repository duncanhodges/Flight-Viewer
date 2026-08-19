# Flight View

A single-file, browser-based visualiser for drone flight logs. Drop in a **GPX** track or a **raw Betaflight / INAV blackbox** file and it renders the flight on a map, plots altitude and speed, draws the 3D path, and produces a printable flight-log entry — all locally in your browser.

- **No install, no build, no server.** It's one self-contained `.html` file.
- **Nothing is uploaded.** Your log is parsed in the browser. The only optional network call is the location lookup (see [Privacy](#privacy)).
- **Reads raw blackbox directly.** No need to convert to CSV/GPX first — the Betaflight binary decoder is built in.

---

## Contents

- [What it shows](#what-it-shows)
- [Quick start](#quick-start)
- [Supported files](#supported-files)
- [Betaflight logging requirements](#betaflight-logging-requirements)
- [How it works](#how-it-works)
- [Flight-reference overlays (120 m / 500 m)](#flight-reference-overlays-120-m--500-m)
- [The logbook & CSV export](#the-logbook--csv-export)
- [Privacy](#privacy)
- [Hosting on GitHub Pages](#hosting-on-github-pages)
- [Dependencies & attribution](#dependencies--attribution)
- [Limitations](#limitations)
- [License](#license)

---

## What it shows

- **Map** — the flight path drawn on a dark, light, or satellite basemap, coloured by altitude (a viridis gradient with a legend). Start and end are marked, and a **500 m range ring** is drawn around the take-off point.
- **Altitude profile** — altitude vs. distance, with a **dual vertical scale**: metres above sea level (MSL) on the left and metres above the take-off point on the right, plus a dashed **120 m above take-off** reference line.
- **Speed** — ground speed vs. distance, with a **dual m/s and mph** scale. Derived over a short centred window so high-rate GPS jitter doesn't produce nonsense spikes.
- **3D path** — a rotatable, zoomable view of the track in longitude/latitude/altitude.
- **Stat strip** — distance, duration, max altitude, max height above launch, top speed, and more.
- **Flight log** — a printable summary (date, take-off/landing, duration, craft, location, max height/distance/speed) with **Print / Save PDF** and **Save CSV**.

Hovering the altitude or speed charts drops a marker on the map at that point, so you can trace exactly where a climb, dive, or fast run happened.

---

## Quick start

1. Open `flight-view.html` in a modern browser (Chrome, Edge, Firefox, Safari), **or** visit the GitHub Pages URL if you've deployed it.
2. Drag a `.gpx`, `.bfl`, or `.bbl` file onto the drop zone — or click to browse.
3. The flight renders. Click **Flight log** (top right) for the printable summary and CSV export.

That's it. There is no account, upload, or configuration step.

---

## Supported files

| Format | Extension | Notes |
| --- | --- | --- |
| GPX track | `.gpx` | Standard `<trkpt>` points with `<ele>` and `<time>`. Also reads `<rtept>`/`<wpt>` as a fallback. |
| Betaflight / INAV blackbox | `.bfl`, `.bbl` | Raw binary log, decoded in-browser. Validated against Betaflight 4.5.x, blackbox **data version 2**. |

The app auto-detects which format you dropped, so you don't need to tell it.

---

## Betaflight logging requirements

The blackbox file only contains a usable flight track if the flight controller was **logging GPS data during the flight**. To get that, four things need to be in place on the aircraft.

### 1. A working GPS

- In **Betaflight Configurator → Ports**, assign a spare UART to **GPS** and set the matching baud rate for your module.
- In **Configuration** (or the **GPS** tab in newer Configurator versions), enable **GPS**, choose the protocol your module speaks (e.g. UBLOX), and configure the home/arming behaviour.
- Before flying, **wait for a 3D fix** with enough satellites. GPS frames are only meaningful once the receiver has locked on — a log recorded with no fix will have no track.

> GPS position is written to the blackbox automatically whenever a GPS is present and providing data. There is no separate "log GPS" switch to enable — a configured, locked GPS is the requirement.

### 2. Blackbox logging enabled

- Enable the **Blackbox** feature in **Configuration**.
- In the **Blackbox** tab, choose a logging device: onboard **dataflash** chip, an **SD card**, or an external serial logger. Make sure there's free space.
- Set a logging rate. GPS frames are logged at a low rate (typically a few hertz) regardless of the main loop rate, so even a modest blackbox rate captures the full track.
- Logging starts when you **arm** (with the usual "log on arm" behaviour).

### 3. Craft name and clock (for the logbook)

These aren't required to draw the flight, but they populate the flight-log sheet:

- **Craft name** — set it in **Configuration → Personalization** (or the Setup/OSD craft-name field). It becomes the aircraft label and the first part of the exported CSV filename.
- **Real-time clock** — the log's start date/time comes from the flight controller's RTC, which Betaflight sets **from GPS time** once it has a fix (or from the Configurator when connected). With a GPS fix, you get an accurate UTC timestamp for the flight.

### 4. Retrieve the log

- Pull the log with **Betaflight Configurator → Blackbox → download**, or copy the `LOGxxxxx.BFL` / `.BBL` file off the SD card.
- Drop that file straight into Flight View. No decoding step required.

---

## How it works

Everything runs client-side. The page is plain HTML/CSS/JavaScript with two libraries loaded from a CDN ([Leaflet](https://leafletjs.com/) for the map, [Plotly](https://plotly.com/javascript/) for the charts).

### GPX parsing

GPX is XML, so it's parsed with the browser's `DOMParser`. Track points are read for latitude, longitude, elevation, and time, then cumulative ground distance and 3D distance are computed with the haversine formula.

### Blackbox decoding

Betaflight's blackbox format is a compact, delta-encoded binary stream — not something a browser can read out of the box. Flight View includes a decoder written from the open blackbox format that:

1. **Parses the header** to learn each frame type's field names, signedness, predictors, and encodings.
2. **Walks the binary frames** in order (intra-frames `I`, inter-frames `P`, GPS frames `G`, GPS-home frames `H`, slow frames `S`, and events `E`), decoding every field with the correct variable-byte and tagged-group encodings so the stream stays byte-aligned.
3. **Reconstructs GPS positions.** Coordinates are stored as deltas from a logged home point and altitudes/speeds as their own encodings; the decoder applies the per-field predictors to rebuild absolute values, and anchors frame timestamps to the log's start datetime.

The decoder's GPS output was checked **row-for-row against the reference `blackbox_decode`** across full logs — latitude, longitude, altitude, timestamp, satellite count, speed, and course all matched exactly. Only GPS-relevant data is surfaced; the decoder reads the rest of the stream purely to stay aligned.

### Map & charts

Points are fed into Leaflet (a per-segment coloured polyline plus the range ring and markers) and Plotly (altitude, speed, and 3D). The charts share the same underlying points, which is what lets hovering one highlight the matching spot on the map.

### Location lookup

On request, the launch coordinates are reverse-geocoded to a UK postcode (nearest postcode → nearest outward code) and a place name. See [Privacy](#privacy).

---

## Flight-reference overlays (120 m / 500 m)

Two reference overlays are drawn relative to each flight's take-off point:

- **120 m above take-off** — a dashed line on the altitude profile, matching the right-hand "above take-off" scale.
- **500 m range** — a ring on the map around the launch position.

These correspond to common UK operating limits (a 120 m / 400 ft maximum height and a typical visual-line-of-sight range). They are **visual references only** — they are not legal advice, and the height figure is measured **above the take-off point**, not above the terrain further out. Always check and follow the rules that apply where you fly.

---

## The logbook & CSV export

The **Flight log** button opens a printable sheet with the flight's date, take-off/landing times (shown in UTC and your local time), duration, craft name, launch position, max height above launch, max altitude, max distance from launch, top speed, distance flown, and the firmware/board from the log header.

- **Print / Save PDF** uses the browser's print dialog (choose "Save as PDF").
- **Save CSV** writes a one-row logbook record you can stack into a master spreadsheet. The filename is built as:

  ```
  <craft>_<YYYYMMDD>_<HHMM>_<outcode>.csv
  ```

  using the craft name, the local flight date, the local take-off time (hours/minutes), and the outward half of the nearest postcode — so repeat flights from the same site on the same day never collide. If no location was looked up, the postcode part is simply omitted.

---

## Privacy

Log parsing, rendering, the logbook, and CSV export all happen **entirely in your browser**. Your flight file is never sent anywhere.

The **one** feature that uses the network is **Look up location**, which sends only the launch coordinates to:

- [postcodes.io](https://postcodes.io/) — nearest UK postcode / outward code, and
- [OpenStreetMap Nominatim](https://nominatim.org/) — a place name.

It is opt-in (a button), and the logbook panel says so. Nothing else about your flight leaves the browser. Outside the UK the postcode lookup will simply return no match and fall back to a place name where available.


---

## Dependencies & attribution

- **[Leaflet](https://leafletjs.com/)** — map rendering.
- **[Plotly.js](https://plotly.com/javascript/)** — altitude, speed, and 3D charts.
- **Basemap tiles** — © OpenStreetMap contributors, © CARTO (dark/light), and Esri/Maxar (satellite).
- **[postcodes.io](https://postcodes.io/)** and **[OpenStreetMap Nominatim](https://nominatim.org/)** — optional reverse geocoding.
- The blackbox decoder was validated against Betaflight's reference **[blackbox-tools](https://github.com/betaflight/blackbox-tools)**.

Please respect each service's usage and attribution terms if you deploy this publicly.

---

## Limitations

- **Firmware coverage.** The decoder is validated against Betaflight 4.5.x, blackbox **data version 2**. Other/older firmware or unusual field sets may not decode; if a log can't be read, the app says so rather than showing garbage, and you can fall back to a GPX exported by the Configurator or `blackbox_decode`.
- **GPS only.** The decoder surfaces the GPS track (position, altitude, time). It does not currently expose gyro, PID, motor, RPM, or battery channels — although they are all present in the file.
- **Height is launch-relative.** "Above take-off" and the 120 m line use height above the launch point as an AGL proxy. Over sloping or coastal terrain, true height above ground further from launch will differ.
- **GPS altitude.** Altitude comes from the GPS module and carries the usual GPS vertical error; it is not a barometric or terrain-referenced height.
- **Postcode lookup is UK-focused.** It falls back to a place name elsewhere.
