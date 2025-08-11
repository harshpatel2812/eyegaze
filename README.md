Optimouse is an accessibility-focused Python application that enables hands-free computer control using eye tracking (via a webcam) and voice commands.
It combines MediaPipe for facial landmark detection, OpenCV for video processing, PyAutoGUI for mouse control, and SpeechRecognition for natural voice interactions.

#How It Works

*Eye Tracking

>The webcam detects your facial landmarks using MediaPipe.

>The average position of your left eye is mapped to the screen.

>Blinking triggers a click event.

*Voice Control 

>Continuously listens for activation keywords.

>On activation, processes the spoken command and executes the matching action.

>Provides audio feedback for confirmation.


🗂️ Voice Commands List

*Command Example	Action Performed

>"click"	Mouse click

>"scroll up"	Scroll up

>"scroll down"	Scroll down

>"open browser"	Open Google homepage

>"search for cats"	Search Google for "cats"

>"open youtube"	Open YouTube

>"increase volume"	Set system volume to 70%

>"mute volume"	Mute system volume

>"lock computer"	Lock workstation

>"stop program"	Exit program
