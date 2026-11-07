# Instrument Translation

A computer-vision project exploring a virtual piano played on paper using a phone camera, OpenCV, and MediaPipe.

## Requirements

- Python 3.12
- `pip`
- A phone with a camera and browser
- Phone and computer on the same Wi-Fi (for the [local network](#option-1-local-network-no-ngrok) setup),
  or [ngrok](https://ngrok.com/download) as an alternative

## Setup

Clone the repository:

```powershell
git clone <repository-url>
cd Instrument-Translation
```

Make sure Python 3.12 is selected as the project interpreter. In VS Code, use
**Python: Select Interpreter** and choose the Python 3.12 interpreter (or the
project's `.venv` after creating it). In PowerShell, use `py -3.12` to create
the environment with Python 3.12 even if the `python` command points to a
different installation.

Create and activate a virtual environment:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

After activation, confirm the project environment is being used:

```powershell
python -c "import sys; print(sys.executable)"
```

The printed path should point to `.venv\Scripts\python.exe`.

Install Python dependencies:

```powershell
python -m pip install -r requirements.txt
```

### macOS setup

The steps above are for Windows. On a Mac, install Python 3.12 with
[Homebrew](https://brew.sh) (newer Python versions are not supported by MediaPipe),
then create the environment from the repo root:

```bash
brew install python@3.12
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

`python -c "import sys; print(sys.executable)"` should print a path ending in
`.venv/bin/python`. In VS Code, select the `.venv` interpreter. Activate the
environment again (`source .venv/bin/activate`) in every new terminal.

The MediaPipe Hand Landmarker model is included in `models/hand_landmarker.task`.

Next, follow [Option 1: Local network](#option-1-local-network-no-ngrok) to run it.

## Running

Phone browsers only allow camera access over HTTPS, so the phone page has to be
served over HTTPS. There are two ways to do that:

- **[Local network](#option-1-local-network-no-ngrok)**: the phone opens your
  computer's IP address directly (e.g. `https://10.19.172.29:5000`). No extra
  software; this is the tested setup on macOS.
- **[ngrok](#option-2-ngrok)**: a public HTTPS tunnel. Use this if your Wi-Fi
  blocks devices from reaching each other.

### Option 1: Local network (no ngrok)

The server serves HTTPS itself using a self-signed certificate. The phone and
computer must be on the **same Wi-Fi**, and the network must let devices reach
each other (many school and public networks block this; use ngrok there).

1. Create a certificate once, from the repo root (on macOS `openssl` is built in;
   on Windows run it from Git Bash):

   ```bash
   openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
     -keyout key.pem -out cert.pem -subj "/CN=localhost"
   ```

   `*.pem` is gitignored. Never commit `key.pem`.

2. Start the server with `--https` (with the `.venv` activated):

   ```bash
   python phone_server.py --https
   ```

3. Find your computer's local IP address:
   - macOS: `ipconfig getifaddr en0` (prints something like `10.19.172.29`)
   - Windows: `ipconfig`, look for the IPv4 address

4. On your phone, open `https://<that-ip>:5000`, for example
   `https://10.19.172.29:5000`. Include `https://` and `:5000`.
5. The phone warns that the certificate isn't trusted. That's expected, so continue
   (Safari: **Show Details → visit this website**; Chrome: **Advanced → Proceed**).
6. Select **Start Camera** and allow camera access.
7. Press `Q` in the OpenCV window to stop the server.

The IP address can change when you switch networks, so check it again with step 3
if the page stops loading. The certificate expires after 365 days; rerun step 1 to
make a new one.

#### macOS troubleshooting

- `No module named cv2`: the system Python is running instead of the `.venv`.
  Run `source .venv/bin/activate` first, or use `.venv/bin/python phone_server.py --https`.
- Port 5000 already in use: turn off **AirPlay Receiver** in
  **System Settings → General → AirDrop & Handoff**.
- Page won't load on the phone: check both devices are on the same Wi-Fi and the
  IP address is current.

### Option 2: ngrok

Install and configure ngrok using its [official documentation](https://ngrok.com/docs/)
(on macOS: `brew install ngrok`).

1. Start the server (without `--https`):

   ```powershell
   python phone_server.py
   ```

2. Open a separate terminal and start the HTTPS tunnel:

   ```powershell
   ngrok http 5000
   ```

3. Open the HTTPS forwarding URL displayed by ngrok on your phone.
4. Select **Start Camera** and allow camera access.
5. Press `Q` in the OpenCV window to stop the server.

Keep both the Python server and ngrok running while using the camera.

## Project Structure

- `phone_server.py` — Flask server, frame processing, and hand tracking.
- `phone_camera.html` — Phone camera interface.
- `phone_camera.css` — Interface styling.
- `phone_camera.js` — Camera capture and frame transmission.
- `paper_detector.py` — Paper detection.
- `models/hand_landmarker.task` — MediaPipe hand-tracking model.
- `requirements.txt` — Python dependencies.

## Using the iPhone over USB (no ngrok)

Install **Camo Camera** (iPhone) and **Camo Studio** (Windows, camo.com), plug in the cable, and open both. Then:

```powershell
python phone_server.py --list-cameras    # find Camo's index
python phone_server.py --camera 1        # use that index
```

The free Camo tier (720p) is enough. Without `--camera`, the phone page + ngrok setup above still works.

Tips:
- Use the iPhone's **1×** lens, not 0.5× (the ultra-wide is softer, bends the paper's edges, and makes keys tiny).
- Keep Camo Camera open and the phone unlocked (Auto-Lock → Never while testing); Camo Studio can stay minimised.
- Run one server at a time, from the project `.venv` (select it as VS Code's interpreter).
- Webcam frames are shrunk to 640 px wide for speed. The second status line shows `rx` (frames arriving), `proc` (ms to process one) and `age` (ms from arrival to ready).
- Plug the laptop in and use the **Best performance** power mode: on low battery the CPU throttles and `proc` can jump past 100 ms.
- Q or Ctrl+C quits cleanly.

## Calibration

The template is black solid boxes or black outlines on white Letter paper, landscape.

- Press `C` with your hands out of view; regions are numbered left to right.
- Press `S` to save the current frame to `samples/`.
- `python calibrate_image.py samples/<file>.jpg` writes a `_debug.png` showing the
  flattened paper, the ink mask and the numbered regions.
- The **Calibration settings** window has **Sens** (sensitivity) (raise it if lines are missed, lower it if
  paper texture or shadows become regions) and **Key** (min black key) (lower it for small drawings).
  Changes re-detect instantly.

## Tap Strap (M0)

`tapstrap/` reads raw finger accelerometer data from a Tap Strap 2 (`python stream_raw.py --demo` for a live readout without a strap).

- `python m0_record.py` runs a prompted session (clicks cue each tap) and writes `recordings/m0_<time>_{fingers,imu,cues}.csv`; the columns are documented in the script. Add `--demo` to try it without a strap.
- `python m0_analyze.py recordings/m0_<time>` writes `_report.txt` and `_report.png` (stream health, per-finger separability, crosstalk, a simple detector's precision/recall).

## Tests

Run the unit tests from the repo root:

```
python -m unittest discover -s tests -t .
```

## Notes

- Phone camera access requires HTTPS; use the ngrok forwarding URL or `--https`
  (see [Local network](#option-1-local-network-no-ngrok)).
- Do not commit `.venv/`, `*.pem` certificate files, authentication tokens, or other secrets.
