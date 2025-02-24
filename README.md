# misbehavior-report
import os
import cv2
import numpy as np
import face_recognition
import tkinter as tk
import threading
from twilio.rest import Client

# Twilio Credentials
TWILIO_ACCOUNT_SID = "AC71f5beec3a66f556b4e012502a6a76df"
TWILIO_AUTH_TOKEN = "615f1b3d1d770ecf3791d673fabf1ce3"
TWILIO_WHATSAPP_NUMBER = "whatsapp:+14155238886"

client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)

# Define HOD contacts
HOD_CONTACTS = {
    "ECE": "whatsapp:+917981259825",
    "CSM": "whatsapp:+916302408766",
}

# Create directory for student images
image_dir = r"C:\Users\narag\OneDrive\Pictures\image\face\images"
os.makedirs(image_dir, exist_ok=True)

# Load known student images & encode faces
known_face_encodings = []
known_face_names = []
students_data = {
    "anjali": {"ID": "12345", "Name": "Anjali", "Branch": "CSM"},
    "khyathi": {"ID": "12346", "Name": "Khyathi", "Branch": "ECE"},
    "snigdha": {"ID": "12347", "Name": "Snigdha", "Branch": "ECE"},
    "vinya": {"ID": "12348", "Name": "Vinya", "Branch": "CSM"},
    "abhisha": {"ID": "12349", "Name": "Abhisha", "Branch": "CSM"},
    "nandini": {"ID": "12349", "Name": "Nandini", "Branch": "ECE"},
}

for student in students_data.keys():
    image_path = os.path.join(image_dir, f"{student}.jpg")
    if os.path.exists(image_path):
        student_image = face_recognition.load_image_file(image_path)
        student_encoding = face_recognition.face_encodings(student_image)
        if student_encoding:
            known_face_encodings.append(student_encoding[0])
            known_face_names.append(student)

# Capture & Save Student Image
def capture_student_image(name, frame):
    image_path = os.path.join(image_dir, f"{name}.jpg")
    cv2.imwrite(image_path, frame)
    return image_path

# Send WhatsApp Message
def send_whatsapp_message(student_info):
    branch = student_info["Branch"]
    hod_number = HOD_CONTACTS.get(branch)
    
    if not hod_number:
        print(f"No HOD contact found for branch: {branch}")
        return

    message_body = (
        f"🚨 Student Misbehavior Report 🚨\n\n"
        f"ID: {student_info['ID']}\n"
        f"Name: {student_info['Name']}\n"
        f"Branch: {student_info['Branch']}\n"
        f"Action Required!"
    )

    try:
        message = client.messages.create(
            body=message_body,
            from_=TWILIO_WHATSAPP_NUMBER,
            to=hod_number
        )
        print(f"WhatsApp message sent to {branch} HOD successfully!")
    except Exception as e:
        print(f"Error sending WhatsApp message: {e}")

# Button to Report Student
def create_report_button(student_info):
    def run_tkinter():
        root = tk.Toplevel()
        root.title("Report Student")
        root.attributes('-topmost', True)  # Ensure it stays on top

        label = tk.Label(root, text=f"Report {student_info['Name']}?", font=("Arial", 12))
        label.pack(pady=10)
        
        btn = tk.Button(root, text="Send Report", command=lambda: send_whatsapp_message(student_info),
                        padx=20, pady=10, bg="red", fg="white")
        btn.pack(pady=10)
        
        root.mainloop()
    
    threading.Thread(target=run_tkinter, daemon=True).start()

# Initialize Webcam
video_capture = cv2.VideoCapture(0)

# Maintain a set to track recognized students
recognized_students = set()

print("Press 'q' to quit the system.")

while True:
    ret, frame = video_capture.read()
    if not ret:
        print("Error: Unable to access the camera.")
        break

    small_frame = cv2.resize(frame, (0, 0), fx=0.25, fy=0.25)
    rgb_small_frame = cv2.cvtColor(small_frame, cv2.COLOR_BGR2RGB)

    face_locations = face_recognition.face_locations(rgb_small_frame)
    face_encodings = face_recognition.face_encodings(rgb_small_frame, face_locations)

    face_names = []
    for face_encoding in face_encodings:
        if known_face_encodings:
            matches = face_recognition.compare_faces(known_face_encodings, face_encoding)
            face_distances = face_recognition.face_distance(known_face_encodings, face_encoding)

            if len(face_distances) > 0:
                best_match_index = np.argmin(face_distances)
                if matches[best_match_index]:
                    name = known_face_names[best_match_index]
                    face_names.append(name)

    for (top, right, bottom, left), name in zip(face_locations, face_names):
        top *= 4
        right *= 4
        bottom *= 4
        left *= 4

        cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)

        if name in students_data and name not in recognized_students:
            recognized_students.add(name)
            student_info = students_data[name]
            
            # Capture and save the student image
            capture_student_image(name, frame)
            
            # Display the recognized student's name on the screen
            cv2.putText(frame, f"{student_info['Name']} - {student_info['Branch']}", 
                        (left + 6, bottom - 10), cv2.FONT_HERSHEY_DUPLEX, 0.6, (255, 255, 255), 1)

            # Open the report button on a separate thread immediately
            threading.Thread(target=create_report_button, args=(student_info,), daemon=True).start()

    cv2.imshow("Student Details", frame)

    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

video_capture.release()
cv2.destroyAllWindows()
print("System closed.")
