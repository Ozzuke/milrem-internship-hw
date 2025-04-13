# UGV HMI Prototype for Milrem Internship

## Features

* **Map Integration:** Displays an OpenStreetMap map using Leaflet.js.
* **UGV Display:** Shows the UGV's current location and direction.
* **Starting Location:** UGV initializes at Milrem office in Tartu.
* **Engine Control:**
    * A visually distinct button allows starting/stopping the mock engine.
    * Movement is disabled if the engine is OFF.
    * A popup notification appears if movement keys are pressed while the engine is OFF.
* **Keyboard Movement (Engine ON):**
    * `ArrowUp`: Move forward in the current heading direction.
    * `ArrowDown`: Move backward.
    * `ArrowLeft`: Rotate UGV counter-clockwise.
    * `ArrowRight`: Rotate UGV clockwise.
* **Camera Lock:** A toggle button allows locking the map view to keep the UGV centered during movement.
* **Waypoint System:**
    * **Add:** Long-press (click and hold) on the map to create a temporary waypoint.
    * **New Waypoint Popup:** Appears after long-press with options:
        * `Drive`: Instantly moves the UGV to the waypoint location and pans the map.
        * `Save`: Saves the waypoint (with an optional name) to a runtime list.
        * `Discard`: Closes the popup without saving.
    * **Waypoint List:** Displays saved waypoints at the bottom-left.
    * **Manage Waypoint Popup:** Appears when clicking a waypoint in the list, with options:
        * `Delete`: Removes the waypoint.
        * `Rename`: Allows changing the waypoint name.
        * `Drive`: Instantly moves the UGV to the waypoint location and pans the map.

## Technology Stack

* Vue 3 (Composition API)
* TypeScript
* HTML5 & CSS3
* Leaflet.js
* `@vue-leaflet/vue-leaflet`
* `leaflet-rotatedmarker`
* Vite (Build Tool / Dev Server)
* Node.js & npm

## Setup Instructions

1. **Prerequisites:** You must have Node.js and npm installed.
2. **Clone:** Clone this repository to your local machine:
   ```bash
   git clone https://github.com/Ozzuke/milrem-internship-hw
   ```
3. **Navigate:** Change into the project directory:
   ```bash
   cd milrem-internship-hw
   ```
4. **Install Dependencies:**
   ```bash
   npm install
   ```
5. **Run Development Server:**
   ```bash
   npm run dev
   ```
6. **Access:** Open your web browser and navigate to the local URL provided by Vite (usually `http://localhost:5173` if no other process uses that port).

## Usage Instructions

1. **Start Engine:** Click the "Start Engine" button (top-right, rocket icon) to enable movement controls. Can also be toggeled with `E` or `Enter`.
2. **Move UGV:** Use the keyboard arrow keys (Up, Down, Left, Right) as described in the Features section.
3. **Lock Camera:** Click the "Lock Cam" button (next to the engine button) to keep the UGV centered while moving. Can also be toggled with `C`.
4. **Add Waypoint:** Click and hold the mouse button down on the desired map location.
5. **Interact with Popups:** Use the buttons within the waypoint popups to Drive to, Save, Discard, Rename, or Delete waypoints.
6. **View Saved Waypoints:** The list appears at the bottom-left once waypoints are saved. Click an item to manage it.

## Most Difficult/Time-Consuming Part

The most time-consuming aspect of this assignment involved debugging specific Leaflet.js integration challenges within
the Vue 3 Composition API environment. Key difficulties included:

* Ensuring the UGV marker's position (`latLng`) and rotation (`rotationAngle`) updated reactively and reliably.
* Correctly implementing the coordinate system logic for movement (e.g., mapping heading angle to `deltaLat`/`deltaLng`
  and applying latitude correction for consistent perceived speed).
* Handling map focus issues after user interaction (like panning) to ensure keyboard controls remained active.

## AI Usage Acknowledgement

AI assistance (GitHub Copilot and Gemini 2.5 Pro) was utilized during development for the following purposes:

* Initial research and comparison between potential mapping libraries (Leaflet was chosen).
* Generating initial boilerplate code for some Vue components.
* Assisting in debugging Leaflet-specific errors, reactivity problems, and focus management issues.
* Refining the movement physics logic, including implementing latitude correction.

## Demo Video

A demonstration video `showcase.mp4` showcasing the application's functionality is included in the root directory of this repository.
