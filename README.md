# Projet-CarBot-Int-gration-de-syst-mes-de-navigation-bio-inspir-s-sur-robot# Polarized Camera Control Software

A PyQt5 graphical interface to control a **LUCID Vision Labs** polarized camera (Arena SDK), display a live video feed, capture images in several polarization formats, and run scheduled (timed) acquisition series.

The software computes the **Degree of Polarization (DoP)**, the **Global Angle of Polarization (AOPG)** and the **Local Angle of Polarization (AOPL)**, with an optional estimation of the solar azimuth using a Hough transform on the AoLP map.

## Table of Contents

- [Features](#features)
- [Interface Overview](#interface-overview)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Capture Formats](#capture-formats)
- [Solar Azimuth Detection (Hough)](#solar-azimuth-detection-hough)
- [Generated Files](#generated-files)
- [Code Structure](#code-structure)
- [Known Limitations](#known-limitations)
- [License](#license)

## Features

- 🔍 **Automatic detection** of the connected LUCID camera (up to 6 attempts, 10 s apart).
- 🎚️ Manual **Gain (dB)** and **Exposure time (µs)** adjustment via sliders.
- 🎥 **Live video stream** (Mono12) that can be started/stopped on demand.
- 📸 **Single image capture** in 4 formats: Raw, AOPG, AOPL, DoP.
- ⚙️ **Adaptive auto-exposure** (iterative adjustment of exposure, then gain, until the mean brightness converges to a target value).
- 🔁 **Series acquisition**: capture of RAW (Mono12) images over a configurable total duration and interval, with automatic saving (8-bit PNG + 12-bit MAT) and a progress bar.
- 🖱️ **Image saving** via right-click on any displayed capture.
- 🌞 **Solar azimuth estimation** using a Hough transform on pixels whose AoLP is close to 90°, with a dedicated OpenCV visualization.
- 🎨 Results displayed in a 3×3 grid, each capture paired with a color bar (colormap).

## Interface Overview

The application is organized into:
- a **sidebar**: list of detected cameras + settings panel (gain, exposure, single capture, series acquisition);
- a main tabbed area: **Live view** (live stream), **Result** (single captures), **Acquisition** (image series).

## Requirements

- **Python** 3.8+ (recommended)
- A **LUCID Vision Labs** polarized camera supporting the `Mono12`, `BayerRG12`, `PolarizedDolp_BayerRG8` and `PolarizedDolpAolp_BayerRG8` pixel formats
- LUCID Vision Labs' **Arena SDK** installed (drivers + `arena_api` Python module)
- Operating system: Windows (the code references `ProgramData\Lucid Vision Labs\...` paths) or Linux with the Arena SDK installed

### Python Dependencies

```
PyQt5
numpy
opencv-python
matplotlib
scipy
```

> `arena_api` (LUCID SDK) is not available on PyPI: it must be installed from the package provided by LUCID Vision Labs with the Arena SDK.

## Installation

1. Install the **Arena SDK** and the `arena_api` Python module provided by LUCID Vision Labs (see the vendor documentation).
2. Clone this repository:
   ```bash
   git clone <repo-url>
   cd <repo-name>
   ```
3. Create a virtual environment (optional but recommended):
   ```bash
   python -m venv venv
   source venv/bin/activate      # Linux/macOS
   venv\Scripts\activate         # Windows
   ```
4. Install the dependencies:
   ```bash
   pip install PyQt5 numpy opencv-python matplotlib scipy
   ```
5. Connect the LUCID camera and verify it is recognized by the Arena SDK (e.g. via ArenaView).

## Usage

Run the application:

```bash
python Code_PC_LUCID.py
```

On startup, the program looks for a connected camera (up to 6 attempts, 10 seconds apart). Once the camera is detected:

1. Adjust **Gain** and **Exposure time** if needed.
2. Click **Start** (Live view) to display the live stream, **Stop** to stop it.
3. Choose an **image format** from the drop-down list, then click **Capture** for a single capture (*Result* tab).
4. For a series of images: set the **total duration**, the **interval**, select a **save directory**, then click **Start** in the acquisition section (*Acquisition* tab). **Stop** interrupts the running series.
5. **Right-click** on any captured image to save it to disk.

## Capture Formats

| Format | Camera pixel format | Processing | Output |
|---|---|---|---|
| **Raw Image** | `BayerRG12` / `Mono12` | Simplified demosaicing (averaging of green pixels), 8-bit normalization | Grayscale image |
| **Degree of Polarization (DoP)** | `PolarizedDolp_BayerRG8` | Viridis colormap | RGB image + colorbar (0–1) |
| **Global Angle of Polarization (AOPG)** | `PolarizedDolpAolp_BayerRG8` | DoLP/AoLP split, HSV colormap, Hough-based solar azimuth detection | RGB image + colorbar (0–180°) |
| **Local Angle of Polarization (AOPL)** | `PolarizedDolpAolp_BayerRG8` | AOPG − geometric angle φ(x,y) computed from the camera's optical center (Scaramuzza-type calibration) | RGB image (HSV colormap) + colorbar (0–180°) |

**Auto-exposure** (`auto_expose`) automatically adjusts the exposure time (and, if needed, the gain) through successive iterations, measuring the mean brightness of a central patch of the image, until it converges to a target value (with configurable tolerance and maximum number of iterations).

## Solar Azimuth Detection (Hough)

The `detect_sun_line_hough` function isolates pixels whose AoLP is close to 90° (perpendicular to the sun's direction), applies a morphological dilation, then a **probabilistic Hough transform** (`cv2.HoughLinesP`) to identify the dominant neutral line. The solar azimuth is derived from the angle of this line (with an offset, `AZIMUT_OFFSET`, accounting for the camera axis orientation relative to North).

The `visualiser_hough` function displays (in a dedicated OpenCV window) the mask of the relevant pixels, the detected line, and the estimated azimuth.

## Generated Files

During a **series acquisition**, for each image captured in the selected folder:

- `N_Raw_image_YYYYMMDD_HHMMSS_G:<gain>_E:<exposure>.png` — 8-bit image (preview)
- `N_Raw_image_YYYYMMDD_HHMMSS.mat` — raw 12-bit data (`ImageMono12`) + metadata (`Gain_dB`, `Exposure_us`), MATLAB format (via `scipy.io.savemat`)

During a **single capture**, the image is displayed in the interface and can be saved manually via the right-click context menu (*Save image*).

## Code Structure

- `detect_sun_line_hough(...)` / `visualiser_hough(...)` — solar azimuth estimation and visualization
- `ClickableLabel` — `QLabel` with right-click image saving
- `ArenaLikeGUI` (`QMainWindow`) — main window:
  - Camera initialization (`system.create_device`, `select_device`)
  - Settings panel (gain, exposure, sliders)
  - `lancer_previsualisation` / `afficher_flux` / `arreter_previsualisation` — live stream
  - `auto_expose` — adaptive auto-exposure
  - `capture_image` — single capture (Raw / DoP / AOPG / AOPL)
  - `lancer_acquisition` / `arreter_acquisition` — scheduled series acquisition
  - `rafraichir_grille_resultats` — grid display of results
  - `select_save_path` — save directory selection

## Known Limitations

- The **optical calibration** parameters (`x_centre`, `y_centre` used in the AOPL computation) are hard-coded and **specific to a given camera** (Scaramuzza-type calibration) — must be adapted for any other camera.
- The referenced main file name (`Code_PC_LUCID.py`) should be adjusted to match the actual script name in the repository.
- The code contains paths and comments referring to **ArenaView / Jupyter** examples (LUCID Vision Labs) that some sections are based on.
- The Hough-based solar azimuth detection opens a separate OpenCV window (`cv2.imshow`), independent of the PyQt interface.
- Tested with a single connected camera at a time (the code selects a single device).

## License

*(to be completed according to the license chosen for this project)*
