# TransactionTroubles Authoring And Hosting Guide

## 1. Overview

This document explains how the interactive “TransactionTroubles” experience is structured and how non-technical editors can add or update slides by editing a single JSON file (`slides.json`). 

It also explains what is required to host or view the app, including the limitations of opening it directly from a desktop file system.

### Download

The entire app can be downloaded from:

https://github.com/VirtuosiMedia/epipheo/archive/refs/heads/main.zip

---

## 2. Files That Control The Experience

The project is a static web app made of:

* **`index.html`** – The main HTML file. It contains the splash screen and loads the JavaScript and styles. 
* **`js/app.js`** – The engine that loads the slide configuration, renders images and videos, and handles overlays, audio, and navigation. 
* **`js/slides.json`** – The content file. All paths (Bank, Meal, Travel) and all slides are defined here. 
* **`media/...`** – All images, videos, icons, and audio files referenced from `slides.json`.

The app engine reads `js/slides.json` at startup via an HTTP `fetch` call, then renders the appropriate scenes. 

---

## 3. Top-Level JSON Structure

`slides.json` is a single JSON object with one main property: `paths`.

```json
{
  "paths": [
    {
      "title": "Bank",
      "image": "media/splash/bank.png",
      "slides": [ /* slide objects */ ]
    },
    {
      "title": "Meal",
      "image": "media/splash/meal.png",
      "slides": [ /* slide objects */ ]
    }
  ]
}
```

* **`paths`** – Array of “adventures” available on the splash screen.
* Each **path** has:

  * `title` – Text label; used mainly for semantics, not directly shown in `slides.json` examples.
  * `image` – Splash thumbnail shown as the clickable card.
  * `slides` – Array of slide objects that play in order when that path is selected.

---

## 4. Defining A Path

A new path is added by creating a new object inside the `paths` array:

```json
{
  "title": "New Scenario",
  "image": "media/splash/new_scenario.png",
  "slides": [
    /* slides go here */
  ]
}
```

**Key points:**

* `image` should point to a PNG/JPG inside the `media/splash/` folder (or another existing folder), using a path relative to `index.html`.
* `slides` must be a non-empty array; otherwise the path will end immediately.

---

## 5. Slide Structure: The Basics

Each slide is an object inside a path’s `slides` array. At minimum, a slide has:

* `base` – The main image or video.
* `advance` – How the slide ends: `"click"`, `"timer"`, or `"video-end"`.
* `preloadNext` – Helps the app load the next slide’s media early.
* `overlays` – Optional list of extra UI elements (arrows, thought bubbles, CTA boxes, audio triggers, etc.).

Example of a simple slide:

```json
{
  "base": {
    "type": "image",
    "src": "media/ui/overlays/UI_bank-01.jpg",
    "alt": "Bank frame 02"
  },
  "advance": "click",
  "preloadNext": true,
  "overlays": []
}
```

---

## 6. Base Media Options

The `base` object defines the primary content of the slide.

### 6.1 Base Image

```json
"base": {
  "type": "image",
  "src": "media/ui/overlays/UI_bank-03.jpg",
  "alt": "Bank frame showing dashboard"
}
```

* `type`: must be `"image"`.
* `src`: relative path to an image file.
* `alt`: short description for screen readers.

### 6.2 Base Video

```json
"base": {
  "type": "video",
  "src": "media/bank/bank_intro.mp4",
  "alt": "Bank intro video frame",
  "caption": "Intro walkthrough",      // optional label for accessibility
  "controls": false,                   // optional: show native video controls if true
  "poster": "media/bank/poster.png"    // optional static preview frame
}
```

Recognized base video fields (as used by `app.js`):

* `type`: `"video"`.
* `src`: video file path.
* `alt`: alternative text (used conceptually similarly to `caption`).
* `caption`: optional description applied as `aria-label` for assistive tech. 
* `controls`: `true` or `false` (default: `false`).
* `poster`: optional image path to show before playback.

---

## 7. Slide Advancement Behavior

The `advance` property controls when the slide ends and the next slide begins.

### 7.1 `advance: "click"`

The slide waits until the viewer clicks in the stage area (or uses keyboard keys such as Space/Enter/Right Arrow) to move to the next slide. 

```json
"advance": "click"
```

This is the most common setting in the current content. 

### 7.2 `advance: "timer"`

The slide automatically advances after a certain number of milliseconds. A `duration` field must be provided.

```json
"advance": "timer",
"duration": 5000   // 5000 ms = 5 seconds
```

### 7.3 `advance: "video-end"`

When the base is a video, the slide can wait until the video finishes playing before advancing.

```json
"base": {
  "type": "video",
  "src": "media/travel/travel_outro.mp4",
  "alt": "Travel outro"
},
"advance": "video-end"
```

### 7.4 Interaction With Overlays

The engine disables stage-level click-to-advance if any overlay on that slide is interactive (a button/hotspot with `action`). This prevents a single click from both firing an overlay and skipping the slide immediately. 

---

## 8. Preloading Behavior

`preloadNext` is a boolean:

```json
"preloadNext": true
```

* When `true`, the engine preloads the next slide’s base media (and overlay media) in the background. 
* Keeping this as `true` provides smoother transitions, especially for videos.

---

## 9. Overlays: Position, Timing, And Persistence

Overlays are additional elements drawn on top of the base (icons, thought bubbles, CTA panels, sound triggers, etc.). Overlays are stored in the `overlays` array.

### 9.1 Position And Size (Percentage-Based Box)

Most overlays use relative units based on percentages of the stage:

* `x` – Left offset as a percentage of the stage width.
* `y` – Top offset as a percentage of the stage height.
* `w` – Width as a percentage.
* `h` – Height as a percentage.

```json
{
  "x": 10,
  "y": 20,
  "w": 10,
  "h": 10
}
```

`x = 10` means 10% from the left edge; `y = 20` means 20% from the top, and so on. 

### 9.2 Timing Properties

Overlays support several timing controls:

* `delay` (ms):

  * For **sequential overlays** with `autoAdvance: true`, `delay` is the “dwell time” before the flow automatically moves to the next overlay.
  * For non-autoAdvance overlays in a **sequential flow**, `delay` is a **pre-show delay** after a click.
  * Outside sequential flows (original behavior via `renderOverlays`), `delay` is often used for scheduling sound-only overlays; some existing content uses `delay` to schedule appearance when no `showAt` is provided.
* `showAt` (ms): Time after slide start when the overlay becomes visible.
* `hideAt` (ms): Time after slide start when the overlay becomes hidden.

Example:

```json
{
  "type": "image",
  "src": "media/ui/overlays/arrow.svg",
  "x": 85,
  "y": 3,
  "w": 10,
  "h": 10,
  "delay": 2000
}
```

Example with explicit show/hide:

```json
{
  "type": "button",
  "action": "skip",
  "persistent": true,
  "x": 88,
  "y": 3,
  "w": 10,
  "h": 10,
  "showAt": 2000,
  "hideAt": 8000
}
```

### 9.3 Persistence

* `persistent: true` means the overlay is not part of the sequential “step-through” flow; it stays visible (subject to optional timing) while other overlays appear and disappear.
* Persistent overlays are used for skip buttons or static elements such as CTA boxes and thought bubble graphics.

---

## 10. Sequential Overlay Flows

Some slides use **sequential overlays**, meaning overlays appear one at a time with each click, before the slide itself advances. 

The engine uses sequential logic when:

* `advance` is `"click"`.
* The slide has overlays.
* `sequential` is **not** explicitly set to `false`.

On such slides:

* Sequential overlays are the non-persistent, non-sound ones.
* Each click reveals the next overlay.
* Once the last overlay is shown, a final click advances the slide.

To **disable sequential behavior** and use pure time-based overlays, set:

```json
"sequential": false
```

Example (Travel path slide that uses time-based overlays instead of sequential stepping): 

```json
{
  "base": {
    "type": "image",
    "src": "media/ui/overlays/UI_airline-03.jpg",
    "alt": "Travel frame 02"
  },
  "advance": "click",
  "preloadNext": true,
  "sequential": false,
  "overlays": [
    ...
  ]
}
```

---

## 11. Overlay Types

### 11.1 Image Overlay

Shows an image at the specified position:

```json
{
  "type": "image",
  "src": "media/ui/overlays/continue.svg",
  "x": 88,
  "y": 3,
  "w": 10,
  "h": 10,
  "delay": 2000
}
```

Fields:

* `type`: `"image"`.
* `src`: image path.
* `alt`: optional alt text.
* Position/timing fields: `x`, `y`, `w`, `h`, `delay`, `showAt`, `hideAt`.
* May also include audio arrays (`music` / `sound`), described later.

### 11.2 HTML Overlay

Renders arbitrary HTML (paragraphs, links, buttons, etc.) inside the overlay box.

```json
{
  "type": "html",
  "persistent": true,
  "x": 0,
  "y": 0,
  "w": 100,
  "h": 100,
  "html": "<div class='cta-overlay'><p>Want to learn more about Digital Experience Analytics?</p><p>Check out <a target=\"_blank\" href=\"https://splunk.com/o11y\">Splunk Observability Website</a> to learn more.</p></div>",
  "delay": 0
}
```

* `html`: raw HTML string inserted into the slide.
* Links (`<a href="...">`) work normally; the overlay’s event handlers prevent clicks on links from accidentally advancing the slide. 
* Often marked `persistent: true` for long-lived UI (CTA overlays, explanatory text).

Another example (CTA button):

```json
{
  "type": "html",
  "persistent": true,
  "x": 0,
  "y": 33,
  "w": 100,
  "h": 100,
  "html": "<button class=\"cta-button\">Try another scenario</button>",
  "delay": 0
}
```

### 11.3 Text Overlay

`type: "text"` works similarly to HTML, but wraps the string in a `<div>`:

```json
{
  "type": "text",
  "x": 8,
  "y": 10,
  "w": 40,
  "h": 20,
  "html": "<p class=\"thought-text\">Looks like the drop offs are increasing along with user count.</p>",
  "persistent": true,
  "showAt": 500
}
```

The current content uses `type: "html"` for most text bubbles, but `type: "text"` is also supported.

### 11.4 Video Overlay (Embedded Video Window)

Displays a video inside a subsection of the slide, often simulating a UI video player.

```json
{
  "type": "video",
  "src": "media/travel/travel_ui_loop.mp4",
  "x": 38.7,
  "y": 20,
  "w": 54,
  "h": 71,
  "autoplay": true,
  "controls": false,
  "loop": false,
  "delay": 0
}
```

Fields:

* `type`: `"video"`.
* `src`: video path.
* `autoplay`: `true` to start playback when visible.
* `loop`: `true` to loop; `false` to play once.
* `controls`: not exposed as native controls in the overlay implementation (play is controlled by overlay click + an optional play button).
* `playLabel`: optional accessible label for the play button.

Overlay video specifics:

* The overlay wrapper is clickable and shows a play button element by default.
* Clicking inside the overlay starts playback and hides the play button.
* When `autoplay` is `true`, playback begins immediately and the play button is hidden. 

### 11.5 Button / Hotspot (Interactive Click Area)

These use `type` such as `"button"` or `"hotspot"`, combined with an `action` field that tells the engine what to do when the overlay is clicked.

Example used as a “skip intro” hotspot:

```json
{
  "type": "button",
  "action": "skip",
  "persistent": true,
  "x": 88,
  "y": 3,
  "w": 10,
  "h": 10,
  "showAt": 2000,
  "hideAt": 65000
}
```

Supported `action` values:

* `"skip"` – Immediately move to the next slide when clicked.
* `"next"` – In a sequential overlay flow, advance to the next overlay (or slide if no more overlays). If no sequential flow is active, behaves like a normal slide advance. 

### 11.6 Sound-Only Overlay

A special overlay type used only to schedule audio playback; it does not draw anything on screen.

```json
{
  "type": "sound",
  "sound": [
    {
      "src": "media/sound/mp3/MusicLoops/SplunkBTM_UI_Music_LOOP.mp3",
      "loops": true,
      "stopOthers": true
    }
  ],
  "delay": 0
}
```

* `type`: `"sound"`.
* `sound`: array of sound entries (see audio section below).
* `delay`: time in ms before the sound begins.

Sound-only overlays are excluded from the visual sequential overlay flow and only control audio timing. 

---

## 12. Audio On Slides And Overlays

The engine distinguishes between **music** (background tracks) and **sound effects**, and keeps track of which are currently playing. 

### 12.1 Music Entries

Music entries are usually defined on overlays via a `music` array:

```json
"music": [
  {
    "src": "media/sound/mp3/MusicLoops/SplunkBTM_UI_Music_LOOP.mp3",
    "loop": true,
    "stopOthers": true
  }
]
```

Each entry supports:

* `src`: audio file path.
* `loop`: `true` for looping (default `true` if omitted).
* `stopOthers`: `true` to stop other music tracks first (default `true` if omitted). 

### 12.2 Sound Effect Entries

Sound entries are used for short effects:

```json
"sound": [
  {
    "src": "media/sound/mp3/InteractiveSounds/thought.mp3",
    "loops": false,
    "stopOthers": false
  }
]
```

Each entry supports:

* `src`: audio file path.
* `loops`: `true` to loop (default `false`).
* `stopOthers`: `true` to stop other sound effects first (default `false`). 

Both `music` and `sound` arrays can appear on any overlay type (image, html, text, etc.). When the overlay becomes visible (or when its scheduled time is reached), the engine starts the configured audio. 

### 12.3 Global Audio Toggle

The app displays a global audio button in the top-right corner (created by `createGlobalAudioToggles`). When audio is disabled there:

* All music and sound are paused.
* All videos are muted. 

This behavior is automatic; no configuration changes in `slides.json` are needed.

---

## 13. Common Editing Tasks

### 13.1 Adding A New Slide To An Existing Path

1. Locate the path in `slides.json` (e.g., the `"Travel"` path).
2. Append a new slide object to the `slides` array.

Example new slide added at the end:

```json
{
  "base": {
    "type": "image",
    "src": "media/ui/overlays/UI_airline-new-step.jpg",
    "alt": "Travel extra explanation frame"
  },
  "advance": "click",
  "preloadNext": true,
  "overlays": [
    {
      "type": "html",
      "persistent": true,
      "x": 5,
      "y": 75,
      "w": 90,
      "h": 20,
      "html": "<p class=\"thought-text\">Extra context about this problem appears here.</p>",
      "delay": 0
    }
  ]
}
```

### 13.2 Changing The Arrow Or CTA Position

To adjust an existing arrow overlay:

```json
{
  "type": "image",
  "src": "media/ui/overlays/arrow.svg",
  "x": 84,   // move horizontally
  "y": 65,   // move vertically
  "w": 10,
  "h": 10,
  "delay": 2000
}
```

Only `x` and `y` need to change in most cases; the rest can remain as is.

### 13.3 Updating Thought Bubble Text

Example from the Meal path: 

```json
{
  "type": "html",
  "persistent": true,
  "x": 8,
  "y": 10,
  "w": 40,
  "h": 20,
  "html": "<p class=\"thought-text\">Looks like I can see related entities to check the backend and figure out what's going on.</p>",
  "showAt": 500
}
```

Updating the text is done by editing the `html` string between the `<p>` tags, while leaving the structure, classes, and timing unchanged.

### 13.4 Updating CTA Text Or Link

Example CTA overlay:

```json
{
  "type": "html",
  "persistent": true,
  "x": 0,
  "y": 0,
  "w": 100,
  "h": 100,
  "html": "<div class='cta-overlay'><p>Want to learn more about Digital Experience Analytics?</p><p>Check out <a target=\"_blank\" href=\"https://splunk.com/o11y\">Splunk Observability Website</a> to learn more.</p></div>",
  "delay": 0
}
```

* To change the text, modify the text inside `<p>...</p>`.
* To change the target page, update the `href` attribute inside the `<a>` tag.

---

## 14. Hosting And Viewing The App

### 14.1 Why Direct File Opening May Not Work

The app loads `js/slides.json` via `fetch("js/slides.json")`. 

When `index.html` is opened directly from the filesystem (a `file://` URL), many browsers block `fetch` for local files due to security rules. In that case:

* Slides will not load.
* The splash screen may show a loading message or an error.

Because of this, the app needs to be served over **HTTP** (even if only on a local machine).

### 14.2 Basic Requirements

To run correctly, the following conditions are needed:

* A modern web browser (Chrome, Edge, Firefox, Safari).
* All project files kept together with the same folder structure:

  * `index.html`
  * `js/app.js`
  * `js/slides.json`
  * `media/...` (with all images, videos, and audio referenced in the JSON).
* A simple HTTP server that serves the project folder.

### 14.3 Local Viewing With Python’s Built-In HTTP Server

On a machine with Python 3 installed, a simple local server can be started using Python’s standard library:

1. Place all project files in a folder, for example `transactiontroubles/`.

2. Open a terminal or command prompt.

3. Change directory to that folder:

   ```bash
   cd path/to/transactiontroubles
   ```

4. Run one of the following commands:

   **Python 3:**

   ```bash
   python3 -m http.server 8000
   ```

   or, on some systems:

   ```bash
   python -m http.server 8000
   ```

5. In a browser, open:

   * `http://localhost:8000/`

The browser will load `index.html`, and `fetch("js/slides.json")` will succeed because the file is being served over HTTP from the local server.

To stop the server, the terminal window can be closed or the process can be interrupted (for example, by pressing `Ctrl + C` in the terminal).

### 14.4 Deploying To A Web Server Or Static Hosting

Any static web server capable of serving HTML, JavaScript, and media files can host the app:

1. Upload the full folder structure to the server:

   * Preserve all relative paths (`js/`, `media/`, etc.).
2. Ensure the entry URL points to `index.html` in the project folder.
3. Confirm that `js/slides.json` is reachable at the path expected in `app.js` (currently `js/slides.json`). 

This setup works with common static hosting setups (corporate intranet servers, cloud object storage configured for website hosting, or general static hosting services) as long as the files are served over HTTP(S).

---

## 15. JSON Reference Summary

This section summarizes the main configuration fields for quick reference.

### 15.1 Path Object

```json
{
  "title": "Bank",
  "image": "media/splash/bank.png",
  "slides": [ /* Slide[] */ ]
}
```

* `title`: Text label.
* `image`: Splash card image path.
* `slides`: Array of slide objects.

### 15.2 Slide Object

```json
{
  "base": { /* BaseMedia */ },
  "advance": "click" | "timer" | "video-end",
  "duration": 3000,        // only if advance = "timer"
  "preloadNext": true,
  "sequential": true,      // optional; default auto-enabled for click + overlays
  "overlays": [ /* Overlay[] */ ]
}
```

### 15.3 BaseMedia

Image base:

```json
{
  "type": "image",
  "src": "media/ui/overlays/UI_bank-01.jpg",
  "alt": "Bank frame 02"
}
```

Video base:

```json
{
  "type": "video",
  "src": "media/bank/bank_intro.mp4",
  "alt": "Bank intro frame",
  "caption": "Intro video",
  "controls": false,
  "poster": "media/bank/poster.png"
}
```

### 15.4 Overlay (Common Fields)

```json
{
  "type": "image" | "html" | "text" | "video" | "button" | "hotspot" | "sound",
  "x": 0,
  "y": 0,
  "w": 100,
  "h": 100,
  "persistent": true,
  "showAt": 1000,
  "hideAt": 5000,
  "delay": 0,
  "action": "next" | "skip",
  "classList": ["optional-css-class"],
  "music": [ /* MusicEntry[] */ ],
  "sound": [ /* SoundEntry[] */ ]
}
```

### 15.5 MusicEntry

```json
{
  "src": "media/sound/mp3/MusicLoops/SplunkBTM_UI_Music_LOOP.mp3",
  "stopOthers": true,
  "loop": true
}
```

### 15.6 SoundEntry

```json
{
  "src": "media/sound/mp3/InteractiveSounds/thought.mp3",
  "loops": false,
  "stopOthers": false
}
```

---

By editing `js/slides.json` using the patterns above and preserving the existing file/folder structure, additional scenarios and slides can be created or modified while reusing the existing engine, audio, and overlay behaviors implemented in `app.js`.
