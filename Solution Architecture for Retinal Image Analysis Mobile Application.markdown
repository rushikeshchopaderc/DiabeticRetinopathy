# Revised Solution Architecture: Retinal Image Analysis Mobile Application on Google Cloud

## Overview

This revised architecture incorporates an account activation and password setup email service for doctors using Cloud Pub/Sub, additional authentication filters to prevent duplicate doctors and patients, and the updated BigQuery schema for `Doctors`, `Patients`, and `Records` tables. The application allows doctors to register, authenticate, upload retinal images for patients, run them through an object detection model on Vertex AI, provide feedback on inference results, and store data in BigQuery. Images are stored in GCS, and the system includes CI/CD, MLOps for model retraining, and monitoring/governance.

---

## Architecture Components

### 1. Mobile Application (Frontend)

- **Platform**: Native (iOS/Android) or cross-platform (e.g., Flutter/React Native).
- **Features**:
  - Doctor registration with email-based activation and password setup.
  - Doctor authentication (login/logout).
  - Patient management (add/view patients).
  - Capture/upload retinal images.
  - Display analysis results (positive/negative for retinal damage).
  - Provide feedback on inference results (correct/incorrect).
  - View historical retinal images, results, and feedback.
- **Integration**:
  - Communicates with backend APIs (Cloud Functions).
  - Uses Firebase Authentication SDK for secure login.

### 2. Authentication

- **Service**: Firebase Authentication with Cloud Functions for custom logic.
- **Details**:
  - Doctors register using their email as `user_id`.
  - **New Filters**:
    - **Doctor Exists Check**:
      - Cloud Function (`register_doctor`) queries BigQuery to check if a doctor with the same `user_id` (email) exists.
      - If exists, return an error: "Doctor already exists."
    - **Patient Exists Check**:
      - Cloud Function (`add_patient`) queries BigQuery to check if a patient with the same `patient_id` exists for the doctor.
      - If exists, return an error: "Patient already exists."
  - Firebase Authentication generates JWT tokens for API requests.
  - Tokens are validated by backend services.

### 3. Account Activation and Password Setup (New)

- **Services**: Cloud Pub/Sub, Cloud Functions, SendGrid (or Gmail API via Cloud Functions).
- **Details**:
  - **Registration Process**:
    1. Doctor submits registration request (email as `user_id`, name) via mobile app.
    2. Cloud Function (`register_doctor`) checks if the doctor exists in BigQuery.
    3. If not, creates a Firebase Authentication user (disabled initially) and writes doctor data to BigQuery.
    4. Publishes a message to a Cloud Pub/Sub topic (`account-activation`).
  - **Activation Workflow**:
    1. Cloud Function (`send_activation_email`) is triggered by the Pub/Sub topic.
    2. Generates a unique activation token (e.g., UUID) and stores it in BigQuery (`Doctors` table).
    3. Sends an email with:
       - Activation link: `https://<app-domain>/activate?token=<token>`.
       - Password setup link: `https://<app-domain>/set-password?token=<token>`.
    4. Doctor clicks the activation link, which calls a Cloud Function (`activate_account`).
    5. Cloud Function validates the token, enables the Firebase Authentication user, and updates BigQuery.
    6. Doctor clicks the password setup link, sets a password via Firebase Authentication.
  - **Pub/Sub Setup**:
    - Topic: `account-activation`.
    - Subscription: `send-activation-sub`.
  - **Email Service**:
    - Use SendGrid or Gmail API via Cloud Functions to send emails.
    - Example (Python with SendGrid):

      ```python
      import sendgrid
      from sendgrid.helpers.mail import Mail
      from google.cloud import pubsub_v1

      def send_activation_email(event, context):
          message = event['data']
          user_id = message['user_id']
          token = message['token']
          sg = sendgrid.SendGridAPIClient(api_key='<sendgrid-api-key>')
          email = Mail(
              from_email='no-reply@retinal-app.com',
              to_emails=user_id,
              subject='Activate Your Retinal App Account',
              html_content=f'Click to activate: <a href="https://retinal-app.com/activate?token={token}">Activate</a><br>'
                          f'Set Password: <a href="https://retinal-app.com/set-password?token={token}">Set Password</a>'
          )
          sg.send(email)
      ```

### 4. Backend Services

- **Service**: Cloud Functions (serverless, cost-optimized).
- **Functions**:
  - **register_doctor**:
    - Validates if doctor exists in BigQuery.
    - Creates Firebase user (disabled) and publishes to Pub/Sub.
  - **send_activation_email**:
    - Triggered by Pub/Sub, sends activation and password setup email.
  - **activate_account**:
    - Validates token, enables Firebase user, updates BigQuery.
  - **add_patient**:
    - Validates if patient exists in BigQuery.
    - Adds patient to BigQuery.
  - **upload_image**:
    - Handles retinal image uploads, stores in GCS, writes metadata to BigQuery.
  - **process_image**:
    - Triggered by GCS, sends image URI to Vertex AI for inference.
  - **submit_feedback**:
    - Stores doctor feedback (correct/incorrect) in BigQuery.
  - **get_results**:
    - Retrieves results and patient history from BigQuery.
- **API Gateway**:
  - Cloud Endpoints to manage API routing and authentication.

### 5. Image Storage

- **Service**: Google Cloud Storage (GCS).
- **Structure**:
  - Bucket: `retinal-images-<project-id>`.
  - Path: `/doctor_id/patient_id/timestamp_eye.jpg`.
  - **Access Control**:
    - IAM roles for authenticated services.
    - Signed URLs for secure access by the mobile app.
  - **Lifecycle Rules**:
    - Move to Coldline after 1 year.
    - Delete after 7 years.

### 6. Object Detection Model

- **Service**: Vertex AI.
- **Details**:
  - Pre-trained object detection model fine-tuned for retinal damage detection.
  - Deployed on Vertex AI Endpoint for real-time inference.
  - **Workflow**:
    - Cloud Function sends GCS image URI to Vertex AI.
    - Vertex AI returns result (positive/negative).
    - Result and feedback are stored in BigQuery.

### 7. Data Storage (Tabular)

- **Service**: BigQuery.
- **Schema** (Updated per Document):
  - **Table 1: Doctors Table**:

    ```sql
    user_id (STRING): Doctor's email (used as Firebase UID)
    name (STRING): Doctor's name
    activation_token (STRING): Token for account activation (New)
    is_active (BOOLEAN): Account activation status (New)
    created_at (TIMESTAMP): Account creation time
    ```
  - **Table 2: Patients Table**:

    ```sql
    patient_id (STRING): Unique patient ID
    doctor_id (STRING): Foreign key to Doctors (user_id)
    name (STRING): Patient name
    dob (DATE): Date of birth
    created_at (TIMESTAMP): Record creation time
    ```
  - **Table 3: Records Table** (Replaces `retinal_images`):

    ```sql
    record_id (STRING): Unique record ID
    patient_id (STRING): Foreign key to Patients
    doctor_id (STRING): Foreign key to Doctors (user_id)
    image_uri (STRING): GCS path to image
    eye (STRING): "left" or "right"
    timestamp (TIMESTAMP): Image capture time
    result (STRING): "positive" or "negative"
    confidence (FLOAT): Model confidence score
    feedback (STRING): "correct" or "incorrect"
    feedback_timestamp (TIMESTAMP): Feedback submission time
    ```
- **Access Control**:
  - IAM roles for backend services.
  - Row-level security for doctor-specific data.
- **Optimization**:
  - Partition `Records` table by `timestamp`.
  - Cluster by `doctor_id`.

### 8. CI/CD Pipeline

- **Services**: GitHub + Google Cloud Build.
- **Details**:
  - GitHub hosts the codebase for Cloud Functions and mobile app.
  - Cloud Build automates build, test, and deployment on code changes.
  - Trigger: Push/pull request to `main` branch.
  - Deploys Cloud Functions and distributes mobile app via Firebase App Distribution.

### 9. MLOps Pipeline for Model Retraining

- **Services**: Vertex AI Pipelines, Cloud Scheduler, BigQuery, GCS.
- **Details**:
  - **Feedback Collection**:
    - Stored in `Records` table.
  - **Retraining Workflow**:
    1. Cloud Scheduler triggers monthly retraining.
    2. Vertex AI Pipeline queries `Records` table for feedback data.
    3. Exports data to GCS for training.
    4. Retrains model using preemptible VMs.
    5. Deploys new model to Vertex AI Endpoint.
  - **Accuracy Monitoring**:
    - Logs accuracy metrics to Cloud Monitoring.
    - Alerts if accuracy drops below 90%.

### 10. Monitoring and Governance

- **Application Monitoring**:
  - Cloud Monitoring tracks Cloud Functions, GCS, and BigQuery performance.
  - Alerts on high latency, errors, or quota limits.
- **Model Monitoring**:
  - Vertex AI Model Monitoring tracks inference latency, prediction drift.
  - Custom metrics track feedback-based accuracy.
- **Governance**:
  - IAM and Audit Logs for access control and compliance.
  - Vertex AI Model Registry for model versioning.
  - Data retention policies in BigQuery and GCS.

### 11. Workflow Orchestration

- **Services**: Cloud Functions, Cloud Pub/Sub, Cloud Scheduler.
- **Triggers**:
  - **Account Activation** (Pub/Sub):
    - Triggered by registration, sends email.
  - **Image Upload** (Cloud Function):
    - Triggered by GCS, processes image via Vertex AI.
  - **Scheduled Tasks** (Cloud Scheduler):
    - Monthly model retraining.
    - Daily GCS cleanup.

---

## Data Flow

 1. Doctor registers via mobile app (email as `user_id`).
 2. Cloud Function checks if doctor exists in BigQuery; if not, creates Firebase user (disabled) and publishes to Cloud Pub/Sub.
 3. Pub/Sub triggers email with activation and password setup links.
 4. Doctor activates account and sets password via links.
 5. Doctor logs in, adds patient (checked for duplicates), and uploads retinal image.
 6. Image is stored in GCS, metadata in BigQuery (`Records`).
 7. GCS upload triggers inference via Vertex AI.
 8. Result is stored in BigQuery and returned to the mobile app.
 9. Doctor provides feedback, stored in `Records`.
10. Monthly retraining pipeline uses feedback data to update the model.
11. System performance and model accuracy are monitored with alerts.

---

## Scalability

- Cloud Functions, Vertex AI, BigQuery, and GCS scale automatically.
- Vertex AI Pipelines handle large-scale retraining.

---

## Cost Optimization

- Preemptible VMs for Vertex AI training.
- GCS lifecycle rules (Coldline after 1 year, delete after 7 years).
- BigQuery partitioning/clustering for query efficiency.
- Cloud Functions for cost-effective backend.

---

## Deployment Steps

 1. Set up GCP project and enable APIs (Cloud Functions, GCS, BigQuery, Vertex AI, Firebase, Cloud Build, Pub/Sub).
 2. Configure Firebase Authentication and SendGrid for emails.
 3. Create GCS bucket and set lifecycle rules.
 4. Define BigQuery schema (`Doctors`, `Patients`, `Records`).
 5. Deploy object detection model to Vertex AI.
 6. Develop and deploy Cloud Functions.
 7. Set up Cloud Pub/Sub for account activation emails.
 8. Configure GitHub and Cloud Build for CI/CD.
 9. Create Vertex AI Pipeline for retraining.
10. Set up Cloud Monitoring, Model Monitoring, and Audit Logs.
11. Develop mobile app with registration, activation, and feedback features.
12. Test end-to-end flow and monitor performance.

---

## Future Enhancements

- Add real-time notifications for critical results via Cloud Pub/Sub.
- Implement federated learning for privacy-preserving updates.
- Integrate AI-driven insights (e.g., severity scoring) in BigQuery.
- Support multi-language UI in the mobile app.