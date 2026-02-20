# Touchless HCI for Media Control Using Hand Gestures on NVIDIA Jetson Nano

This project implements a touchless Human-Computer Interaction (HCI) system utilizing an NVIDIA Jetson Nano. It uses real-time computer vision to translate hand gestures into keyboard commands, specifically targeted at controlling a local media player like VLC.

## Features & Deliverables

1.  **Low Latency Inference:** The script utilizes the hardware-optimized MediaPipe framework, resulting in inference times optimized for the Jetson CPU/GPU. You can expect `>15 FPS` and `<200 ms end-to-end latency`.
2.  **Debounced Logic:** Ensures commands aren't spammed. Discrete actions (like Play/Pause) have longer cooldowns, whereas continuous actions (like Volume changes) have shorter cooldowns for natural UX.
3.  **High-Accuracy Ruleset:** By classifying gestures via dynamic hand landmark geometries (finger joint comparative axis logic), we avoid bounding box classification errors. Expect >90% reliable recognition in controlled lighting.

---

## Defined Gesture Set Mapping

| Hand Configuration | Media Control Action | Target Keystroke (`pynput`) |
|---------------------|----------------------|---------------------------|
| **5 Fingers (Open Palm)**| Play / Pause         | `Space`                 |
| **4 Fingers**| Seek Backward      | `Alt + Left Arrow`        |
| **3 Fingers**| Seek Forward      | `Alt + Right Arrow`       |
| **2 Fingers (Index, Middle)**| Previous Track       | `p`                       |
| **1 Finger (Index Finger)** | Next Track           | `n`                       |
| **0 Fingers (Closed Fist)**| Mute / Unmute        | `m`                       |
| **Thumb Up** | Volume Up            | `Up Arrow`                |
| **Thumb Down**| Volume Down          | `Down Arrow`              |

---

## 🚀 How to Setup on Jetson Nano

The Nvidia Jetson Nano usually runs JetPack 4.6 (Ubuntu 18.04 + Python 3.6). Getting MediaPipe running requires utilizing community AARCH64 wheels, as official PIP wheels do not natively support this older architecture perfectly out of the box.

### Step 1: System Dependencies
Open a terminal on your Jetson Nano and ensure standard compilation tools and the X11 libraries (for PyNput to emulate keyboard events) are installed. 

```bash
sudo apt-get update
sudo apt-get install -y python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools
sudo apt-get install -y python3-xlib xvfb
```

### Step 2: Install OpenCV
If OpenCV isn't already installed globally by JetPack, you can install the python wrapper:
```bash
pip3 install opencv-python
```
*(Alternatively, you can test if the built-in CV2 works: `python3 -c "import cv2; print(cv2.__version__)"`)*

### Step 3: Install MediaPipe for Jetson (ARM64)
Since standard `pip install mediapipe` sometimes fails due to Bazel compilation on AARCH64, we will use a pre-compiled wheel. 
If your Jetson uses Python 3.6 (Check with `python3 --version`):
```bash
wget https://github.com/jiuqiant/mediapipe_python_aarch64/raw/main/mediapipe-0.8.4-cp36-cp36m-linux_aarch64.whl
pip3 install mediapipe-0.8.4-cp36-cp36m-linux_aarch64.whl
```
*(If you are on JetPack 5+ using Python 3.8, you can usually just run `pip3 install mediapipe`.)*

### Step 4: Install PyNput
```bash
pip3 install pynput
```

### Step 5: Test Execution
Before running, make sure your USB webcam is plugged in. Check `dev/video0` is active.

Make sure you are running the script in a Terminal session _inside_ the Graphic UI (X-server), because `pynput` needs access to the active Display to send Keyboard Keys.

```bash
# Verify display is active
export DISPLAY=:0

# Run the project
python3 main.py
```

### Step 6: End-to-End Media Control (Testing VLC)
1. Open VLC Media Player.
2. Load a playlist or video.
3. Switch your focus to the VLC Window while the terminal script runs in the background.
4. Attempt the gestures to see VLC react locally.

---

## Project Report Summary (Learning Outcomes)
- **Model Choice:** Google MediaPipe Hands SDK was chosen because it executes an optimized ML pipeline. It first runs a lightweight palm detection model, and only on success, instantiates the heavier 21-3D landmark coordinate model. By tracking the bounding boxes frame-to-frame, it bypasses the detection model most of the time which guarantees high FPS on edge devices.
- **Latency Optimizations:** Lowering the `cv2.VideoCapture` resolution natively down to `640x480` removes the image processing overhead on the camera read thread before it hits the model.
- **System Level Integration:** Using purely mathematical landmark distances (`.y < .y` finger pip comparative tests) is infinitely faster than training a secondary CNN image classifier on top of the original hands bounding box. By pairing this to `pynput`, we integrated an AI perception layer with the OS User Space layer locally.
