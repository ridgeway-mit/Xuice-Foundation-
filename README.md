import csv
import cv2
import dropbox
import mediapipe as mp
import numpy as np
import os
from datetime import datetime

# Initialize MediaPipe
mp_pose = mp.solutions.pose
mp_draw = mp.solutions.drawing_utils

# Dropbox configuration (read from environment variable)
DROPBOX_ACCESS_TOKEN = os.getenv("DROPBOX_ACCESS_TOKEN", "")
DROPBOX_FOLDER = "/PoseTracking"  # Folder path in your Dropbox

# Output path
output_dir = os.path.expanduser("~/Desktop/PoseTracking")
os.makedirs(output_dir, exist_ok=True)
timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
output_file = os.path.join(output_dir, f"pose_tracking_{timestamp}.mp4")
landmarks_file = os.path.join(output_dir, f"pose_landmarks_{timestamp}.csv")
print(f"Video output: {output_file}")
print(f"Landmark data output: {landmarks_file}")
if DROPBOX_ACCESS_TOKEN:
    print(f"Dropbox upload enabled to: {DROPBOX_FOLDER}")
else:
    print("Dropbox upload disabled. To enable, set DROPBOX_ACCESS_TOKEN.")

# Phone camera settings
PHONE_CAMERA_INDICES = [1, 0]
PHONE_CAMERA_BACKENDS = [
    ("AVFoundation", cv2.CAP_AVFOUNDATION),
]


def open_camera_candidate(index, backend):
    cap = cv2.VideoCapture(index, backend)
    if not cap.isOpened():
        return None, None
    ret, frame = cap.read()
    if not ret or frame is None:
        cap.release()
        return None, None
    return cap, frame


def choose_phone_camera():
    print("Opening camera previews for candidate indices 1 and 0.")
    print("Press 'y' when the preview shows your iPhone camera, or 'n' to try the next option.")
    print("If you want to stop, press 'q'.")

    for index in PHONE_CAMERA_INDICES:
        backend_name, backend = PHONE_CAMERA_BACKENDS[0]
        cap, frame = open_camera_candidate(index, backend)
        if cap is None:
            print(f"  index={index} failed to open.")
            continue

        label = f"Preview index={index}. Press y for this camera, n for next."
        cv2.putText(frame, label, (10, 40), cv2.FONT_HERSHEY_SIMPLEX, 0.8, (0, 255, 0), 2)
        cv2.imshow("Phone camera preview", frame)

        while True:
            key = cv2.waitKey(0) & 0xFF
            if key == ord('y'):
                cv2.destroyAllWindows()
                print(f"Selected index={index}, backend={backend_name}")
                return cap, index, backend_name
            if key == ord('n'):
                cap.release()
                cv2.destroyAllWindows()
                break
            if key == ord('q'):
                cap.release()
                cv2.destroyAllWindows()
                return None, None, None

    return None, None, None

def open_landmark_csv(path):
    csv_file = open(path, 'w', newline='')
    csv_writer = csv.writer(csv_file)
    header = ['frame']
    for i in range(33):
        header += [f'x_{i}', f'y_{i}', f'z_{i}', f'visibility_{i}']
    csv_writer.writerow(header)
    return csv_file, csv_writer

def upload_to_dropbox(video_path, landmarks_path):
    if not DROPBOX_ACCESS_TOKEN:
        print("Dropbox upload skipped (no token configured)")
        return
    try:
        print("Uploading to Dropbox...")
        dbx = dropbox.Dropbox(DROPBOX_ACCESS_TOKEN)
        
        # Upload video
        video_name = os.path.basename(video_path)
        with open(video_path, 'rb') as f:
            dbx.files_upload(f.read(), f"{DROPBOX_FOLDER}/{video_name}", mode=dropbox.files.WriteMode('overwrite'))
        print(f"Video uploaded: {video_name}")
        
        # Upload landmarks CSV
        landmarks_name = os.path.basename(landmarks_path)
        with open(landmarks_path, 'rb') as f:
            dbx.files_upload(f.read(), f"{DROPBOX_FOLDER}/{landmarks_name}", mode=dropbox.files.WriteMode('overwrite'))
        print(f"Landmarks uploaded: {landmarks_name}")
        
        print("Dropbox upload complete!")
    except Exception as e:
        print(f"Dropbox upload failed: {e}")


def diagnose_cameras():
    print("\nPhone camera diagnostics:")
    for index in range(6):
        for backend_name, backend in PHONE_CAMERA_BACKENDS:
            cap = cv2.VideoCapture(index, backend)
            ok = cap.isOpened()
            ret, _ = cap.read() if ok else (False, None)
            print(f"  index={index}, backend={backend_name}, opened={ok}, read={bool(ret)}")
            cap.release()
    print("If the phone camera does not appear, make sure it is unlocked and trusted by the Mac.")


cap, selected_index, selected_backend = choose_phone_camera()
if cap is None:
    print("Unable to select the phone camera via USB.")
    diagnose_cameras()
    exit()

print(f"Using phone camera index={selected_index}, backend={selected_backend}")

frame_width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
frame_height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
fps = 30
fourcc = cv2.VideoWriter_fourcc(*"mp4v")
out = cv2.VideoWriter(output_file, fourcc, fps, (frame_width, frame_height))
landmark_csv, landmark_writer = open_landmark_csv(landmarks_file)

print("Camera connected! Starting pose preview.")
print("Press 'r' to toggle recording, 's' to save landmarks now, and 'q' to quit.")

recording = True
frame_count = 0
saved_frames = 0

with mp_pose.Pose(min_detection_confidence=0.5, min_tracking_confidence=0.5) as pose:
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret or frame is None:
            print("Failed to read frame from phone camera.")
            break

        rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        results = pose.process(rgb_frame)

        black_background = np.zeros_like(frame)
        if results.pose_landmarks:
            mp_draw.draw_landmarks(black_background, results.pose_landmarks, mp_pose.POSE_CONNECTIONS)

            row = [frame_count]
            for landmark in results.pose_landmarks.landmark:
                row.extend([landmark.x, landmark.y, landmark.z, landmark.visibility])
            landmark_writer.writerow(row)
            saved_frames += 1

        status = "RECORDING" if recording else "PAUSED"
        cv2.putText(black_background, f"{status}", (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.8, (0, 255, 0), 2)
        cv2.putText(black_background, "r=toggle record, s=save now, q=quit", (10, frame_height - 20), cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 2)
        cv2.putText(black_background, f"Phone idx={selected_index}", (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)

        if recording:
            out.write(black_background)

        cv2.imshow("MediaPipe Pose - Phone Camera", black_background)

        key = cv2.waitKey(1) & 0xFF
        if key == ord('q'):
            break
        if key == ord('r'):
            recording = not recording
            print("Recording" if recording else "Recording paused")
        if key == ord('s'):
            landmark_csv.flush()
            print(f"Landmark CSV saved to: {landmarks_file}")

        frame_count += 1
        if frame_count % 30 == 0 and recording:
            print(f"Recorded {frame_count} frames")

cap.release()
out.release()
landmark_csv.close()
cv2.destroyAllWindows()

print("\nSession complete!")
print(f"Video saved to: {output_file}")
print(f"Landmarks saved to: {landmarks_file}")
print(f"Pose frames with landmarks: {saved_frames}")

upload_to_dropbox(output_file, landmarks_file)

