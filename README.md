# MediaPipe Pose Tracker with iPhone Camera

A Python application that uses MediaPipe to track body poses from your iPhone camera via USB. Records videos with skeleton overlays and exports pose landmark data to CSV for analysis.

## Features

- **iPhone USB Camera Support**: Direct USB connection to capture from your iPhone camera
- **Real-time Pose Tracking**: MediaPipe-based full-body pose detection
- **Video Recording**: Records pose tracking with skeleton visualization to MP4
- **Landmark Export**: Saves all 33 pose landmarks (x, y, z, visibility) to CSV
- **Recording Toggle**: Press `r` to pause/resume recording during capture
- **Automatic Dropbox Upload**: Auto-uploads videos and landmark data to your Dropbox
- **Black Background Visualization**: Skeleton overlays on black background for clarity

## Requirements

- Python 3.9+
- macOS (with AVFoundation support)
- iPhone with USB connection and camera access enabled
- Dropbox account (optional, for auto-upload)

## Installation

1. Create a virtual environment:
```bash
python -m venv mp_env
source mp_env/bin/activate
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

## Configuration

### Dropbox Setup (Optional)

To enable automatic upload to Dropbox:

1. Go to https://www.dropbox.com/developers/apps
2. Create a new app with "Full Dropbox" scope
3. Enable `files.content.write` and `files.metadata.write` permissions
4. Generate an access token

5. Set your access token as an environment variable:

**macOS/Linux:**
```bash
export DROPBOX_ACCESS_TOKEN="your_token_here"
python medipipe/take1.py
```

**Or add to your shell profile (~/.zshrc or ~/.bash_profile):**
```bash
echo 'export DROPBOX_ACCESS_TOKEN="your_token_here"' >> ~/.zshrc
source ~/.zshrc
```

All friends running this code will use your Dropbox token (if you share it), so all videos upload to your research folder automatically.

## Usage

1. Connect your iPhone via USB
2. Unlock your iPhone and trust the computer
3. Run the script:
```bash
python medipipe/take1.py
```

4. When prompted, select your iPhone camera from the preview (press `y`)

5. Controls during recording:
   - `r` - Toggle recording on/off
   - `s` - Save landmark CSV immediately
   - `q` - Quit and finalize recording

6. Video and landmarks are saved to `~/Desktop/PoseTracking/`

## Output Files

- `pose_tracking_YYYYMMDD_HHMMSS.mp4` - Video with skeleton overlay
- `pose_landmarks_YYYYMMDD_HHMMSS.csv` - 33 pose landmarks per frame

## Landmark Format

Each row in the CSV contains:
- `frame` - Frame number
- For each of 33 landmarks: `x_i`, `y_i`, `z_i`, `visibility_i` (normalized coordinates 0-1)

## License

MIT
