# Podo Player (PWA Media Player)

Podo Player ("PodoPlayer") is a Progressive Web Application (PWA) designed for media playback. It utilizes modern web technologies to provide a robust video playing experience, featuring category management, device configuration, and potential remote connectivity via MQTT.

## Features

## Tech Stack

- **Frontend**: HTML5, CSS3, JavaScript
- **UI Framework**: Materialize CSS, jQuery UI
- **Video Player**: Video.js, videojs-playlist
- **Data & Storage**: Dexie.js (IndexedDB), Axios (HTTP requests)
- **Communication**: Paho MQTT (WebSockets)
- **Utilities**: Croner (Cron jobs for JS), jsTree (Interactive trees)

## Installation & Setup

Since this is a client-side PWA, it does not require a backend build process (like `npm install` for a Node server), though it relies on external libraries included in the `js/` directory.


## Usage

1.  **Device ID**: On first load, you may be prompted to enter or confirm a **Device ID**. This is used to identify the player instance.
2.  **Playlists/Categories**: Browse the category list on the left to find media content.
3.  **Playback**: Click on items to start playback in the modal player.
4.  **Mobile**: The interface is responsive; on mobile devices, use the bottom navigation bar to switch between Playlist, Settings, and Player views.

## Project Structure

- `index.html`: Entry point. Handles Service Worker registration and browser support checks.
- `sw-installed.html`: The main application interface loaded after Service Worker installation.
- `sw.js`: Service Worker script for caching and offline support.
- `js/`: Contains application logic (`app.js`, `player.js`, `api.js`, `db.js`, `ui.js`, `ws.js`) and libraries.
- `css/`: Stylesheets.
- `manifest.json`: Web App Manifest configuration.

## License

[License details if available, otherwise "Proprietary" or "MIT"]
