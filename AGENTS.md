# AGENTS.md

## Project: Mobile Web Collector for Urban AR Positioning

This document defines the responsibilities, scope, inputs, outputs, and rules for agents working on the **collector** part of the system.

The collector is a **mobile web page** opened on a phone, used to capture images and sensor metadata and send them to a local laptop server over Wi‑Fi for dataset building and testing.

---

## 1. Goal

Build a lightweight mobile web collector that:

- opens the phone camera in browser
- captures still frames manually or automatically
- reads available sensor metadata
- sends images + metadata to a local backend
- helps the operator collect high-quality reference data for one route point / building

The collector is **not** responsible for AR rendering, historical overlays, or final user-facing experience.

---

## 2. Main Use Case

Operator workflow:

1. Open collector page on phone
2. Select or scan:
   - `route_id`
   - `point_id`
   - `building_id`
3. Allow permissions:
   - camera
   - geolocation
   - motion/orientation if available
4. Capture frames:
   - manual mode
   - timed auto-capture mode
5. Upload data to local laptop server
6. Verify uploaded frames from reviewer UI on laptop

---

## 3. Agent Roles

### 3.1 Collector UI Agent
Responsible for the mobile browser UI.

#### Responsibilities
- camera preview
- capture button
- auto-capture controls
- status indicators
- upload progress
- permission state display
- current scene identifiers display

#### Must Provide
- clear and minimal mobile UI
- usable in bright outdoor conditions
- large buttons
- visible capture status

#### Must Not
- implement backend storage logic
- implement heavy image analysis on device
- depend on native app APIs

---

### 3.2 Sensor Agent
Responsible for collecting metadata available in browser.

#### Responsibilities
- geolocation
- heading / compass if available
- device orientation if available
- screen orientation
- timestamp
- device/browser info

#### Inputs
- browser sensor APIs
- permission state

#### Outputs
Metadata object attached to each capture.

#### Notes
Browser support is inconsistent, especially on iPhone/Safari. The agent must degrade gracefully and never block capture if some sensors are unavailable.

---

### 3.3 Capture Agent
Responsible for extracting image frames from the camera stream.

#### Responsibilities
- manual capture
- interval capture
- image compression before upload
- resolution control
- filename/id generation

#### Rules
- default upload resolution should be moderate, e.g. around 1280px long side
- avoid uploading full-resolution images by default
- preserve aspect ratio
- include capture timestamp in metadata

#### Must Not
- silently discard frames without logging reason
- run expensive image processing that affects UX

---

### 3.4 Upload Agent
Responsible for sending data to the local laptop server.

#### Responsibilities
- POST image + metadata to backend
- retry on transient network failure
- queue uploads if connection is unstable
- show upload success/failure state
- prevent duplicate accidental submission where possible

#### Rules
- uploads should work over local Wi‑Fi
- failed uploads should remain visible to operator
- queue should survive temporary page/network instability when practical

---

### 3.5 Session Agent
Responsible for grouping collected data into logical sessions.

#### Responsibilities
- create and track current capture session
- attach identifiers:
  - `route_id`
  - `point_id`
  - `building_id`
  - `session_id`
- track capture counters
- support notes/tags if needed

#### Purpose
Helps backend organize data and later evaluate dataset quality per scene.

---

## 4. Non-Goals

The collector does **not** need to:

- solve camera pose
- perform server-side matching
- render 3D AR overlays
- do semantic segmentation
- be production-polished
- support all browsers equally
- replace a native capture app

---

## 5. Functional Requirements

## 5.1 Camera
- Open back camera if available
- Show live preview
- Support portrait and landscape
- Allow manual capture
- Allow timed capture every N seconds

## 5.2 Metadata
Each capture should include as many of the following as available:

- `session_id`
- `route_id`
- `point_id`
- `building_id`
- `timestamp_client`
- `gps.lat`
- `gps.lng`
- `gps.accuracy`
- `heading`
- `pitch`
- `roll`
- `screen_orientation`
- `device_pixel_ratio`
- `viewport_width`
- `viewport_height`
- `user_agent`

## 5.3 Upload
- Send image and metadata to local backend endpoint
- Report:
  - pending
  - success
  - failed

## 5.4 Operator Feedback
The page should display:
- current IDs
- number of captured frames
- number uploaded successfully
- current GPS accuracy if available
- sensor availability state
- current capture mode

---

## 6. Suggested Data Contract

## 6.1 Request
`POST /capture`

Recommended payload as `multipart/form-data`:

### Fields
- `image`: JPEG file
- `metadata`: JSON string

Example metadata:

{
  "session_id": "sess_2026_03_10_001",
  "route_id": "center_walk_01",
  "point_id": "p_007",
  "building_id": "b_ivanov_house",
  "timestamp_client": 1760000000,
  "gps": {
    "lat": 55.7512,
    "lng": 37.6184,
    "accuracy": 11.5
  },
  "heading": 118.2,
  "pitch": -4.1,
  "roll": 1.3,
  "screen_orientation": "portrait",
  "viewport": {
    "width": 390,
    "height": 844,
    "dpr": 3
  },
  "user_agent": "Mozilla/5.0 ..."
}

## 6.2 Response
{
  "ok": true,
  "capture_id": "cap_000123",
  "stored": true
}

## 7. UI Requirements

### 7.1 Minimum Screen Elements

live camera preview
route/point/building labels
manual capture button
auto-capture toggle
capture interval selector
upload status
permission/sensor status
frame counter

### 7.2 Nice to Have

blur warning
GPS accuracy badge
“move left/right” operator hints
last captured thumbnail
offline/queue indicator


## 8. Reliability Rules

Capture must still work if compass is unavailable
Capture must still work if GPS is inaccurate
Missing metadata must be explicit as null or omitted consistently
Failed uploads must be visible
The app should favor data collection continuity over perfect metadata completeness


## 9. Performance Rules

Keep the page responsive during capture
Compress images before upload
Avoid high-frequency auto-capture by default
Recommended default interval:
0.5s
to
1.5s
Do not store large images in memory longer than needed


## 10. Browser Constraints

Agents must assume:


iOS Safari may restrict motion/orientation without explicit permission
compass data may be unavailable or inconsistent
geolocation can be noisy in urban areas
camera capabilities vary by device
HTTPS may be required for some APIs, except localhost-like development setups depending on environment

The collector must degrade gracefully.

## 11. Suggested Folder / Module Structure

collector/
  src/
    ui/
    camera/
    sensors/
    capture/
    upload/
    session/
    utils/
  public/
  README.md
  AGENTS.md

## 12. Priorities

Implementation priority:


Open camera reliably
Capture JPEG frames
Attach basic metadata
Upload to local backend
Show clear success/failure state
Add auto-capture
Improve operator UX


## 13. Definition of Done

Collector MVP is done when:


operator can open page on phone
camera preview works
manual capture works
captured image is uploaded to laptop backend
metadata includes at least timestamp + route/point/building IDs
optional sensor data is attached when available
operator can collect a usable dataset for one building


## 14. Future Extensions

Possible later additions:


burst mode
keyframe marking
local blur estimation
duplicate frame suppression
simple facade coverage hints
QR-based prefill of route/point/building
session export/import
local caching with retry queue


## 15. Engineering Style

Keep implementation simple
Prefer browser-native APIs
Minimize dependencies
Make failures visible
Optimize for field usability, not architectural purity
Assume unstable sensors and imperfect network
Build for quick iteration on a laptop-first workflow
