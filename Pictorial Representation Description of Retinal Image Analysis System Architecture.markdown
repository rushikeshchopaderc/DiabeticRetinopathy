# Pictorial Representation Description: Retinal Image Analysis System on Google Cloud

## Overview

This description outlines a pictorial representation of the system architecture for the retinal image analysis mobile application, including the input/output flow. The diagram should visually depict the components, their interactions, and the flow of data between them. Use this to create a diagram in a tool like Lucidchart or Draw.io.

---

## Diagram Layout

### Components and Icons
- **Doctor**: Represent as a human icon.
- **Mobile App**: A smartphone icon.
- **Firebase Authentication**: A shield icon (indicating security).
- **Cloud Functions**: A gear icon (indicating processing).
- **Cloud Pub/Sub**: A message envelope icon.
- **Doctor Email**: An email icon.
- **Google Cloud Storage (GCS)**: A bucket icon.
- **Vertex AI Endpoint**: A neural network icon.
- **Vertex AI Pipeline/Training**: A factory icon (indicating model training).
- **BigQuery**: A database icon.
- **Cloud Scheduler**: A clock icon.
- **GitHub**: A GitHub logo.
- **Cloud Build**: A hammer icon (indicating build process).
- **Cloud Monitoring & Logging**: A magnifying glass icon.

### Layout Structure
- Arrange components in a layered structure:
  - **Top Layer**: Doctor and Mobile App (user interaction).
  - **Middle Layers**: Cloud Functions, Firebase Authentication, Cloud Pub/Sub, GCS, Vertex AI, BigQuery (core processing and storage).
  - **Bottom Layer**: Cloud Scheduler, Vertex AI Pipeline, GitHub, Cloud Build, Cloud Monitoring (orchestration, CI/CD, monitoring).
- Use arrows to show the flow of data (inputs/outputs) between components.
- Color-code arrows for clarity:
  - **Blue Arrows**: User-initiated actions (e.g., registration, image upload).
  - **Green Arrows**: System responses (e.g., email sent, inference result).
  - **Orange Arrows**: Background processes (e.g., retraining, CI/CD).

---

## Pictorial Description

### 1. Top Layer: User Interaction
- **Doctor (Human Icon)**:
  - Positioned at the top-left.
  - Connected to the Mobile App with a blue arrow labeled "1. Register (Input: email, name)".
- **Mobile App (Smartphone Icon)**:
  - Positioned at the top-center.
  - Blue arrow to Cloud Functions (`register_doctor`) labeled "2. Registration Request (Output: user_id, name)".
  - Blue arrow from Mobile App to Cloud Functions (`add_patient`) labeled "10. Add Patient (Input: patient_id, name, dob)".
  - Blue arrow to Cloud Functions (`upload_image`) labeled "13. Upload Image (Input: image, patient_id, eye)".
  - Blue arrow to Cloud Functions (`submit_feedback`) labeled "19. Submit Feedback (Input: feedback)".
  - Green arrow from Cloud Functions (`activate_account`) labeled "8. Enable Firebase User (Output: Success)".
  - Green arrow from Cloud Functions (`get_results`) labeled "18. Return Result (Output: result, confidence)".

### 2. Middle Layers: Core Processing and Storage
- **Firebase Authentication (Shield Icon)**:
  - Positioned in the second layer, left side.
  - Green arrow from Cloud Functions (`activate_account`) to Firebase Authentication labeled "8. Enable Firebase User".
  - Blue arrow from Mobile App labeled "9. Login (Input: user_id, password)".
  - Green arrow to Mobile App labeled "9. (Output: JWT Token)".
- **Cloud Functions (Gear Icon)**:
  - Positioned in the second layer, center, as a group of gears (one for each function).
  - **register_doctor**:
    - Blue arrow from Mobile App labeled "2. Registration Request".
    - Blue arrow to BigQuery (`Doctors Table`) labeled "3. Check if Doctor Exists (Input: user_id)".
    - Green arrow from BigQuery labeled "3. (Output: Error if exists)".
    - Green arrow to Cloud Pub/Sub labeled "4. Publish to Pub/Sub (Output: user_id, token)".
  - **send_activation_email**:
    - Green arrow from Cloud Pub/Sub labeled "5. Trigger Email (Input: user_id, token)".
    - Green arrow to Doctor Email labeled "6. Send Email (Output: Activation/Password Links)".
  - **activate_account**:
    - Blue arrow from Doctor Email labeled "7. Activate & Set Password (Input: token)".
    - Green arrow to Firebase Authentication labeled "8. Enable Firebase User".
  - **add_patient**:
    - Blue arrow from Mobile App labeled "10. Add Patient".
    - Blue arrow to BigQuery (`Patients Table`) labeled "11. Check if Patient Exists (Input: patient_id)".
    - Green arrow from BigQuery labeled "11. (Output: Error if exists)".
    - Green arrow to BigQuery labeled "12. Store Patient (Output: Success)".
  - **upload_image**:
    - Blue arrow from Mobile App labeled "13. Upload Image".
    - Green arrow to GCS labeled "14. Store Image (Output: image_uri)".
  - **process_image**:
    - Green arrow from GCS labeled "15. Trigger Inference (Input: image_uri)".
    - Green arrow to Vertex AI Endpoint labeled "16. Send to Vertex AI (Output: result, confidence)".
    - Green arrow to BigQuery (`Records Table`) labeled "17. Store Result (Input: result, confidence)".
  - **submit_feedback**:
    - Blue arrow from Mobile App labeled "19. Submit Feedback".
    - Green arrow to BigQuery labeled "20. Store Feedback (Output: Success)".
  - **get_results**:
    - Green arrow to BigQuery labeled "Query Results".
    - Green arrow to Mobile App labeled "18. Return Result".
- **Cloud Pub/Sub (Message Envelope Icon)**:
  - Positioned in the second layer, right side.
  - Green arrow from Cloud Functions (`register_doctor`) labeled "4. Publish to Pub/Sub".
  - Green arrow to Cloud Functions (`send_activation_email`) labeled "5. Trigger Email".
- **Doctor Email (Email Icon)**:
  - Positioned in the second layer, far right.
  - Green arrow from Cloud Functions (`send_activation_email`) labeled "6. Send Email".
  - Blue arrow to Cloud Functions (`activate_account`) labeled "7. Activate & Set Password".
- **Google Cloud Storage (Bucket Icon)**:
  - Positioned in the third layer, left side.
  - Green arrow from Cloud Functions (`upload_image`) labeled "14. Store Image".
  - Green arrow to Cloud Functions (`process_image`) labeled "15. Trigger Inference".
  - Orange arrow from Vertex AI Pipeline labeled "23. Store Training Data (Output: dataset)".
  - Orange arrow to Vertex AI Training labeled "24. Retrain Model (Input: dataset)".
- **Vertex AI Endpoint (Neural Network Icon)**:
  - Positioned in the third layer, center.
  - Green arrow from Cloud Functions (`process_image`) labeled "16. Send to Vertex AI".
  - Green arrow to Cloud Functions labeled "16. (Output: result, confidence)".
  - Orange arrow from Vertex AI Training labeled "25. Deploy Model (Output: New Endpoint)".
  - Orange arrow to Cloud Monitoring labeled "26. Monitor Model (Input: predictions)".
- **BigQuery (Database Icon)**:
  - Positioned in the third layer, right side.
  - Three sub-boxes for `Doctors`, `Patients`, and `Records` tables.
  - Blue arrows from Cloud Functions (`register_doctor`, `add_patient`) labeled "Check if Exists".
  - Green arrows from Cloud Functions (`add_patient`, `upload_image`, `process_image`, `submit_feedback`) labeled "Store Data".
  - Orange arrow to Vertex AI Pipeline labeled "22. Extract Feedback Data (Input: records)".

### 3. Bottom Layer: Orchestration, CI/CD, Monitoring
- **Cloud Scheduler (Clock Icon)**:
  - Positioned in the bottom layer, left side.
  - Orange arrow to Vertex AI Pipeline labeled "21. Monthly Retraining Trigger".
- **Vertex AI Pipeline/Training (Factory Icon)**:
  - Positioned in the bottom layer, center.
  - Orange arrow from Cloud Scheduler labeled "21. Monthly Retraining Trigger".
  - Orange arrow from BigQuery labeled "22. Extract Feedback Data".
  - Orange arrow to GCS labeled "23. Store Training Data".
  - Orange arrow to Vertex AI Training labeled "24. Retrain Model".
  - Orange arrow from Vertex AI Training to Vertex AI Endpoint labeled "25. Deploy Model".
- **GitHub (GitHub Logo)**:
  - Positioned in the bottom layer, right side.
  - Orange arrow to Cloud Build labeled "27. Push Code (Input: Code Changes)".
- **Cloud Build (Hammer Icon)**:
  - Orange arrow from GitHub labeled "27. Push Code".
  - Orange arrow to Cloud Functions & Mobile App labeled "28. Build/Test/Deploy (Output: Updated App)".
- **Cloud Monitoring & Logging (Magnifying Glass Icon)**:
  - Positioned in the bottom layer, far right.
  - Orange arrow from Vertex AI Endpoint labeled "26. Monitor Model (Output: Metrics, Alerts)".
  - Orange arrow from Cloud Functions labeled "29. Monitor App (Input: Logs, Metrics)".
  - Orange arrow to itself labeled "29. (Output: Alerts)".

---

## Visual Styling

- **Colors**:
  - Use blue for user-initiated actions (e.g., registration, image upload).
  - Use green for system responses (e.g., inference result, email sent).
  - Use orange for background processes (e.g., retraining, CI/CD, monitoring).
- **Arrows**:
  - Solid arrows for direct data flow.
  - Dashed arrows for event-driven triggers (e.g., GCS to Cloud Functions, Pub/Sub to Cloud Functions).
- **Labels**:
  - Label each arrow with the step number, action, and input/output (e.g., "1. Register (Input: email, name)").
- **Component Grouping**:
  - Group Cloud Functions together in a dashed box to indicate multiple functions.
  - Group BigQuery tables (`Doctors`, `Patients`, `Records`) in a single database icon with sub-labels.

---

## How to Create the Diagram

1. **Choose a Tool**: Use Lucidchart, Draw.io, or any diagramming software.
2. **Add Components**: Place each component as described, using the specified icons.
3. **Draw Arrows**: Connect components with arrows, color-coded as described, and label each arrow with the step number and input/output.
4. **Style the Diagram**:
   - Use consistent colors and fonts.
   - Ensure arrows are clear and not overlapping.
   - Add a legend to explain the color coding (blue for user actions, green for system responses, orange for background processes).
5. **Review**: Ensure all components and flows from the architecture are represented.

This pictorial representation will provide a clear visual of the system architecture and data flow, making it easy to understand the interactions and processes involved.