import sys
import os
import cv2
import mediapipe as mp
import pyautogui
import numpy as np
import time
import threading
import webbrowser
import pyttsx3
import speech_recognition as sr
import sounddevice as sd
import soundfile as sf
from datetime import datetime
import platform

if platform.system() == "Windows":
    import ctypes
    import pygetwindow as gw

# ========== GREETING ==========
def get_greeting():
    current_hour = datetime.now().hour
    if 5 <= current_hour < 12:
        return "Good morning"
    elif 12 <= current_hour < 18:
        return "Good afternoon"
    else:
        return "Good evening"

# ========== VOICE ==========
engine = pyttsx3.init()
engine.setProperty('rate', 150)
voices = engine.getProperty('voices')
engine.setProperty('voice', voices[1].id)  # Use female voice

greeting = get_greeting()
engine.say(f"Hello, {greeting}, harsh.")
engine.runAndWait()

VOICE_PROMPTS = {
    "welcome": "Optimouse system activated. Eye tracking and voice commands ready.",
    "voice_ready": "Voice commands are now active.",
    "command_not_recognized": "Command not recognized.",
    "goodbye": "Shutting down system. Goodbye!",
    "computer_locked": "Computer locked.",
    "volume_muted": "Sound muted.",
    "error": "Action failed.",
    "searching": "Searching for",
}

def speak(prompt_key, extra_text=""):
    message = VOICE_PROMPTS.get(prompt_key, prompt_key) + extra_text
    try:
        print(f"🔊: {message}")
    except UnicodeEncodeError:
        print("Speaking:", message)
    engine.say(message)
    engine.runAndWait()

# ========== CONFIG ==========
pyautogui.FAILSAFE = True
screen_width, screen_height = pyautogui.size()
video_capture = cv2.VideoCapture(0)
if not video_capture.isOpened():
    print("❌ Camera error")
    speak("error")
    exit()

# ========== FACE MESH ==========
mp_face_mesh = mp.solutions.face_mesh
face_mesh = mp_face_mesh.FaceMesh(refine_landmarks=True, max_num_faces=1)

# ========== EYE TRACKING ==========
LEFT_EYE = [362, 385, 387, 263, 373, 380]
RIGHT_EYE = [33, 160, 158, 133, 153, 144]
BLINK_THRESHOLD = 0.2
DWELL_THRESHOLD = 1.5

# ========== VOICE ==========
recognizer = sr.Recognizer()
recognizer.energy_threshold = 300
voice_command_active = True
program_running = True
lock = threading.Lock()

# ========== FUNCTIONS ==========
def eye_aspect_ratio(eye_points, landmarks):
    A = np.linalg.norm(np.array([landmarks[eye_points[1]].x, landmarks[eye_points[1]].y]) -
                       np.array([landmarks[eye_points[5]].x, landmarks[eye_points[5]].y]))
    B = np.linalg.norm(np.array([landmarks[eye_points[2]].x, landmarks[eye_points[2]].y]) -
                       np.array([landmarks[eye_points[4]].x, landmarks[eye_points[4]].y]))
    C = np.linalg.norm(np.array([landmarks[eye_points[0]].x, landmarks[eye_points[0]].y]) -
                       np.array([landmarks[eye_points[3]].x, landmarks[eye_points[3]].y]))
    return (A + B) / (2.0 * C)

def lock_workstation():
    if platform.system() == "Windows":
        ctypes.windll.user32.LockWorkStation()
        speak("computer_locked")

def set_volume(level):
    if platform.system() == "Windows":
        try:
            volume = int((level / 100) * 65535)
            ctypes.windll.winmm.waveOutSetVolume(0, volume)
        except:
            speak("error")
    elif platform.system() == "Darwin":
        os.system(f"osascript -e 'set volume output volume {level}'")

def record_and_recognize():
    try:
        duration = 4
        sample_rate = 16000
        # Print without emoji if needed
        try:
            print("🎤 Listening...")
        except UnicodeEncodeError:
            print("[Listening...]")
        audio = sd.rec(int(duration * sample_rate), samplerate=sample_rate, channels=1, dtype='int16')
        sd.wait()
        temp_file = "temp.wav"
        sf.write(temp_file, audio, sample_rate)

        with sr.AudioFile(temp_file) as source:
            audio_data = recognizer.record(source)
            return recognizer.recognize_google(audio_data).lower()
    except Exception as e:
        print(f"❌ Audio error: {e}")
    return None

def control_browser(action, query=None):
    try:
        if platform.system() == "Windows":
            browser = gw.getActiveWindow()
            if browser and any(name in browser.title.lower() for name in ['chrome', 'firefox', 'edge']):
                browser.activate()
                time.sleep(0.2)
        if action == "search":
            webbrowser.open(f"https://www.google.com/search?q={query}", new=2)
            speak("searching", f" {query}")
        elif action == "scroll_up":
            pyautogui.scroll(300)
        elif action == "scroll_down":
            pyautogui.scroll(-300)
    except:
        speak("error")

def process_voice_command(command):
    command = command.lower()
    try:
        print(f"🎤 Command: {command}")
    except UnicodeEncodeError:
        print("Command:", command)
        
    global program_running, voice_command_active

    if "click" in command:
        pyautogui.click()
    elif "scroll up" in command:
        pyautogui.scroll(200)
    elif "scroll down" in command:
        pyautogui.scroll(-200)
    elif "close window" in command:
        pyautogui.hotkey('alt' if platform.system() == 'Windows' else 'command', 'w')
    elif "open browser" in command:
        webbrowser.open("https://www.google.com", new=2)
    elif "search for" in command:
        query = command.replace("search for", "").strip()
        control_browser("search", query)
    elif "open youtube" in command:
        webbrowser.open("https://www.youtube.com", new=2)
    elif "open gmail" in command:
        webbrowser.open("https://mail.google.com", new=2)
    elif "open google maps" in command:
        webbrowser.open("https://www.google.com/maps", new=2)
    elif "play music" in command:
        webbrowser.open("https://music.youtube.com", new=2)
    elif "increase volume" in command:
        set_volume(70)
    elif "decrease volume" in command:
        set_volume(30)
    elif "mute volume" in command or "mute" in command:
        set_volume(0)
        speak("volume_muted")
    elif "lock computer" in command or "lock pc" in command:
        lock_workstation()
    elif "stop program" in command:
        program_running = False
        speak("goodbye")
        exit()
    elif "stop listening" in command:
        voice_command_active = False
    else:
        print(f"❌ Unrecognized command: {command}")
        speak("command_not_recognized")

# ========== THREAD ==========
def voice_command_listener():
    global voice_command_active
    while program_running:
        command = record_and_recognize()
        if command:
            with lock:
                if not voice_command_active and any(kw in command for kw in ["hey google", "okay google", "hey optimouse", "hello"]):
                    speak("voice_ready")
                    voice_command_active = True
                elif voice_command_active:
                    process_voice_command(command)

# ========== VISUALS ==========
def draw_eye_landmarks(frame, landmarks, eye_points, color):
    h, w = frame.shape[:2]
    for i in eye_points:
        x = int(landmarks[i].x * w)
        y = int(landmarks[i].y * h)
        cv2.circle(frame, (x, y), 2, color, -1)

def add_status_text(frame, text, position=(10, 30)):
    cv2.putText(frame, text, position, cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)

# ========== MAIN ==========
def main():
    global program_running
    voice_thread = threading.Thread(target=voice_command_listener, daemon=True)
    voice_thread.start()

    speak("welcome")
    try:
        while program_running:
            ret, frame = video_capture.read()
            if not ret:
                break
            frame = cv2.flip(frame, 1)
            rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            results = face_mesh.process(rgb)

            if results.multi_face_landmarks:
                for face_landmarks in results.multi_face_landmarks:
                    draw_eye_landmarks(frame, face_landmarks.landmark, LEFT_EYE, (0, 0, 255))
                    draw_eye_landmarks(frame, face_landmarks.landmark, RIGHT_EYE, (0, 0, 255))

                    left_eye = np.mean([(face_landmarks.landmark[i].x, face_landmarks.landmark[i].y) for i in LEFT_EYE], axis=0)
                    screen_x = int(left_eye[0] * screen_width)
                    screen_y = int(left_eye[1] * screen_height)
                    pyautogui.moveTo(screen_x, screen_y, duration=0.1)

                    left_ear = eye_aspect_ratio(LEFT_EYE, face_landmarks.landmark)
                    right_ear = eye_aspect_ratio(RIGHT_EYE, face_landmarks.landmark)
                    if left_ear < BLINK_THRESHOLD and right_ear < BLINK_THRESHOLD:
                        pyautogui.click()
                        add_status_text(frame, "CLICK DETECTED!", (50, 80))
                        time.sleep(0.3)

                    add_status_text(frame, f"Left EAR: {left_ear:.2f}", (10, 30))
                    add_status_text(frame, f"Right EAR: {right_ear:.2f}", (10, 60))

            status = "VOICE: ACTIVE" if voice_command_active else "VOICE: INACTIVE"
            add_status_text(frame, status, (10, 90))
            display_frame = cv2.resize(frame, None, fx=0.8, fy=0.8)
            cv2.imshow("Optimouse - Eye + Voice Control", display_frame)
            if cv2.waitKey(1) & 0xFF == ord('q'):
                program_running = False
    except KeyboardInterrupt:
        print("\n🛑 Keyboard interrupt received. Shutting down.")
    finally:
        program_running = False
        video_capture.release()
        cv2.destroyAllWindows()
        speak("goodbye")
        print("✅ System shutdown complete.")

if __name__ == "__main__":
    main()
