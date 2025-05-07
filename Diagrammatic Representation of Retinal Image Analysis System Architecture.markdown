# Diagrammatic Representation: Retinal Image Analysis System on Google Cloud

## Architecture Diagram with Input/Output Flow

Below is an ASCII art representation of the system architecture, showing the components and the flow of data (inputs and outputs) between them.

```
+----------------------------------------------------------+
|                                                          |
|   [Doctor]                                               |
|      |                                                   |
|      | 1. Register (Input: email, name)                  |
|      v                                                   |
|   [Mobile App]                                           |
|   (iOS/Android)                                          |
|      | 2. Registration Request (Output: user_id, name)    |
|      v                                                   |
|   [Cloud Functions: register_doctor]                     |
|      | 3. Check if Doctor Exists (Input: user_id)         |
|      |    (Output: Error if exists)                      |
|      v                                                   |
|   [BigQuery: Doctors Table]                              |
|      | 4. Create Firebase User (Disabled)                |
|      |    Publish to Pub/Sub (Output: user_id, token)     |
|      v                                                   |
|   [Cloud Pub/Sub: account-activation]                    |
|      | 5. Trigger Email (Input: user_id, token)          |
|      v                                                   |
|   [Cloud Functions: send_activation_email]               |
|      | 6. Send Email (Output: Activation/Password Links)  |
|      v                                                   |
|   [Doctor Email]                                         |
|      | 7. Activate & Set Password (Input: token)          |
|      v                                                   |
|   [Cloud Functions: activate_account]                    |
|      | 8. Enable Firebase User (Output: Success)         |
|      v                                                   |
|   [Firebase Authentication]                              |
|      | 9. Login (Input: user_id, password)               |
|      |    (Output: JWT Token)                           |
|      v                                                   |
|   [Mobile App]                                           |
|      | 10. Add Patient (Input: patient_id, name, dob)    |
|      v                                                   |
|   [Cloud Functions: add_patient]                         |
|      | 11. Check if Patient Exists (Input: patient_id)   |
|      |     (Output: Error if exists)                    |
|      v                                                   |
|   [BigQuery: Patients Table]                             |
|      | 12. Store Patient (Output: Success)              |
|      v                                                   |
|   [Mobile App]                                           |
|      | 13. Upload Image (Input: image, patient_id, eye)  |
|      v                                                   |
|   [Cloud Functions: upload_image]                        |
|      | 14. Store Image (Output: image_uri)              |
|      v                                                   |
|   [Google Cloud Storage]                                 |
|      | 15. Trigger Inference (Input: image_uri)         |
|      v                                                   |
|   [Cloud Functions: process_image]                       |
|      | 16. Send to Vertex AI (Output: result, confidence)|
|      v                                                   |
|   [Vertex AI Endpoint]                                   |
|      | 17. Store Result (Input: result, confidence)      |
|      v                                                   |
|   [BigQuery: Records Table]                              |
|      | 18. Return Result (Output: result, confidence)    |
|      v                                                   |
|   [Mobile App]                                           |
|      | 19. Submit Feedback (Input: feedback)             |
|      v                                                   |
|   [Cloud Functions: submit_feedback]                     |
|      | 20. Store Feedback (Output: Success)             |
|      v                                                   |
|   [BigQuery: Records Table]                              |
|      | 21. Monthly Retraining Trigger                   |
|      v                                                   |
|   [Cloud Scheduler]                                      |
|      | 22. Extract Feedback Data (Input: records)        |
|      v                                                   |
|   [Vertex AI Pipeline]                                   |
|      | 23. Store Training Data (Output: dataset)         |
|      v                                                   |
|   [Google Cloud Storage]                                 |
|      | 24. Retrain Model (Input: dataset)                |
|      |    (Output: New Model)                           |
|      v                                                   |
|   [Vertex AI Training]                                   |
|      | 25. Deploy Model (Output: New Endpoint)           |
|      v                                                   |
|   [Vertex AI Endpoint]                                   |
|      | 26. Monitor Model (Input: predictions, feedback)  |
|      |    (Output: Metrics, Alerts)                     |
|      v                                                   |
|   [Cloud Monitoring]                                     |
|                                                          |
|   [GitHub]                                               |
|      | 27. Push Code (Input: Code Changes)              |
|      v                                                   |
|   [Cloud Build]                                          |
|      | 28. Build/Test/Deploy (Output: Updated App)       |
|      v                                                   |
|   [Cloud Functions & Mobile App]                         |
|      | 29. Monitor App (Input: Logs, Metrics)           |
|      |    (Output: Alerts)                              |
|      v                                                   |
|   [Cloud Monitoring & Logging]                           |
|                                                          |
+----------------------------------------------------------+
```

## Component Descriptions

- **Doctor**: End-user interacting with the mobile app.
- **Mobile App**: Frontend for registration, patient management, image uploads, and feedback.
- **Firebase Authentication**: Manages doctor login and account activation.
- **Cloud Functions**:
  - `register_doctor`: Checks for duplicate doctors, creates Firebase user, publishes to Pub/Sub.
  - `send_activation_email`: Sends activation and password setup email.
  - `activate_account`: Validates token, enables Firebase user.
  - `add_patient`: Checks for duplicate patients, stores in BigQuery.
  - `upload_image`: Stores images in GCS, writes metadata to BigQuery.
  - `process_image`: Sends image URI to Vertex AI for inference.
  - `submit_feedback`: Stores feedback in BigQuery.
- **Cloud Pub/Sub**: Decouples registration from email sending.
- **Google Cloud Storage (GCS)**: Stores retinal images and training datasets.
- **Vertex AI Endpoint**: Hosts the object detection model for inference.
- **Vertex AI Pipeline/Training**: Manages model retraining based on feedback.
- **BigQuery**: Stores `Doctors`, `Patients`, and `Records` tables.
- **Cloud Scheduler**: Triggers monthly model retraining.
- **GitHub & Cloud Build**: Manages CI/CD pipeline for application updates.
- **Cloud Monitoring & Logging**: Tracks application and model performance.

## Input/Output Flow Explanation

1. **Registration**:
   - **Input**: Doctor provides email (`user_id`) and name via mobile app.
   - **Output**: Cloud Function checks BigQuery, creates Firebase user, publishes to Pub/Sub.
2. **Account Activation**:
   - **Input**: Pub/Sub message with `user_id` and `token`.
   - **Output**: Email with activation and password setup links.
3. **Activation & Password Setup**:
   - **Input**: Doctor clicks links with `token`.
   - **Output**: Firebase user enabled, password set.
4. **Login**:
   - **Input**: `user_id` and password.
   - **Output**: JWT token for authentication.
5. **Add Patient**:
   - **Input**: `patient_id`, name, dob.
   - **Output**: Error if patient exists, else stored in BigQuery.
6. **Image Upload & Inference**:
   - **Input**: Retinal image, `patient_id`, `eye`.
   - **Output**: Image stored in GCS, result (`positive/negative`, confidence) stored in BigQuery.
7. **Feedback**:
   - **Input**: Doctor’s feedback (`correct/incorrect`).
   - **Output**: Feedback stored in BigQuery.
8. **Model Retraining**:
   - **Input**: Feedback data from BigQuery.
   - **Output**: New model deployed to Vertex AI Endpoint.
9. **CI/CD**:
   - **Input**: Code changes in GitHub.
   - **Output**: Updated Cloud Functions and mobile app.
10. **Monitoring**:
    - **Input**: Logs, metrics, predictions, feedback.
    - **Output**: Alerts on performance issues or model drift.

## Additional Notes

- **Security**: API calls are secured with Firebase JWT tokens. IAM roles restrict access to GCS, BigQuery, and Vertex AI.
- **Scalability**: Cloud Functions, BigQuery, and Vertex AI scale automatically.
- **Cost Optimization**:
  - Preemptible VMs for Vertex AI training.
  - GCS lifecycle rules (Coldline after 1 year, delete after 7 years).
  - BigQuery partitioning/clustering for query efficiency.