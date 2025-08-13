# 🤖 WhatsApp-Driven Google Drive Assistant

[![n8n](https://img.shields.io/badge/Built%20with-n8n-FF6600?style=for-the-badge&logo=n8n.io)](https://n8n.io/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![Twilio](https://img.shields.io/badge/Twilio-F22F46?style=for-the-badge&logo=twilio&logoColor=white)](https://www.twilio.com/)
[![Google Drive](https://img.shields.io/badge/Google%20Drive-4285F4?style=for-the-badge&logo=google-drive&logoColor=white)](https://www.google.com/drive/)
[![Gemini](https://img.shields.io/badge/Gemini-8E77F0?style=for-the-badge&logo=google-gemini&logoColor=white)](https://ai.google.dev/)

A robust n8n workflow that transforms WhatsApp into a powerful, conversational assistant for managing your Google Drive. This assistant understands simple text commands to list, move, delete, and generate AI-powered summaries of your documents, providing a seamless remote-management experience.

---

### ✨ Features

| Command | Description | Example |
| :--- | :--- | :--- |
| 🚀 **LIST** | Lists all files and folders within a specified directory. | `LIST "/My Reports"` |
| 🔄 **MOVE** | Moves a file from a source path to a destination folder. | `MOVE "/Reports/Q1.pdf" "/Archive"` |
| 🗑️ **DELETE** | Initiates a safe, two-step deletion process to prevent accidents. | `DELETE "/Archive/old.txt"` |
| ✅ **CONFIRM DELETE**| Finalizes the deletion after receiving a confirmation prompt. | `CONFIRM DELETE "/Archive/old.txt"` |
| 🧠 **SUMMARY** | Generates an AI-powered summary for every PDF, DOCX, and TXT file in a folder using Google Gemini. | `SUMMARY "/Meeting Notes"` |
| 📈 **AUDIT LOG** | Every successful action is automatically logged to a Google Sheet with a timestamp, user details, and command specifics. | N/A |
| ❓ **HELP** | Displays a comprehensive list of all available commands and their usage. | `HELP` |

### 🎬 Demo Video


https://github.com/user-attachments/assets/47ff293d-d4c4-4c42-a41a-a1fa4fe1af68


**[https://youtu.be/4kzTz2SnuIo]**

*(A brief video demonstrating the assistant in action, showing a WhatsApp conversation and the corresponding changes in Google Drive and the Google Sheets log.)*

---

### 🔧 Technical Architecture & Design

This workflow is designed to be robust, stateless, and extensible, running entirely within a self-hosted n8n instance on Docker.

1.  **Entrypoint**: A single **n8n Webhook** listens for incoming POST requests from the **Twilio Sandbox for WhatsApp**.
2.  **Universal Parser**: A powerful **Function Node** (JavaScript) acts as a central parser. It uses regular expressions to correctly interpret commands and arguments, including handling paths with spaces when enclosed in double quotes (`"`). It outputs a structured JSON object for the rest of the workflow.
3.  **Command Router**: A central **Switch Node** routes the execution to the appropriate command branch based on the output from the parser. This makes the workflow clean and easy to extend with new commands.
4.  **Action Branches**:
    * **Google Drive**: Each branch uses **Google Drive Nodes** (via OAuth2) to perform file system operations. The `Search` operation with custom queries is used extensively to locate specific files and folders.
    * **AI Summarization**: The `SUMMARY` branch processes files in parallel based on their MIME type (`.pdf`, `.docx`, `.txt`). It uses a combination of n8n's internal `Extract from File` node and external APIs (**ConvertAPI** for DOCX) to extract text, which is then passed to a **Google Gemini Node** for summarization.
5.  **Data Integrity Pattern**: A "pass-it-forward" data pattern using **Set** and **Merge** nodes is implemented throughout the workflow. This ensures that essential data (like original file names or sender information) is never lost as the data stream is transformed by intermediate nodes, preventing common `undefined` errors.
6.  **Centralized Reply & Logging**: All successful command branches converge on a single **Twilio Node** to send the final reply and a single **Google Sheets Node** to append to the audit log, following the DRY (Don't Repeat Yourself) principle.

---

### ⚙️ Setup Instructions

Follow these steps to deploy your own instance of the assistant.

### 1. Prerequisites

-   [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running.
-   An active [Twilio account](https://www.twilio.com/try-twilio) with the WhatsApp Sandbox configured.
-   A [Google Cloud Platform](https://console.cloud.google.com/) project.
-   A [Google AI Studio](https://aistudio.google.com/) account to get a free Gemini API key.
-   (Optional but Recommended) A free [ConvertAPI](https://www.convertapi.com/a) account for `.docx` file summarization.
-   [ngrok](https://ngrok.com/download) to expose your local n8n instance to the internet for Twilio.

### 2. Run n8n with Docker

1.  Clone this repository to your local machine.
2.  Open a terminal in the project's root directory.
3.  Run the command: `docker-compose up -d`.
4.  n8n will be available at `http://localhost:5678`.

### 3. Configure Credentials in n8n

Open your n8n instance (`http://localhost:5678`) and go to the **Credentials** tab. Add the following credentials:

-   **Twilio**: Use your Account SID and Auth Token from the Twilio dashboard.
-   **Google Gemini API**:
    * Get your API key from [Google AI Studio](https://aistudio.google.com/).
    * In n8n, add a `Google Gemini API` credential and paste the key.
-   **Google OAuth2 (for Drive & Sheets)**:
    1.  In your Google Cloud project, enable the **Google Drive API** and **Google Sheets API**.
    2.  Go to **Credentials > Create Credentials > OAuth 2.0 Client ID**. Select "Web application".
    3.  In n8n, create a `Google OAuth2 API` credential. Copy the **Redirect URI** that n8n gives you.
    4.  Go back to your Google Cloud credential settings and paste the URI under "Authorized redirect URIs".
    5.  In n8n, add the following **scopes**: `https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/spreadsheets`.
    6.  Save and sign in with your Google account. On the "OAuth consent screen", add your email address as a "Test user".

### 4. Set up the Workflow

1.  Create a new, blank workflow in n8n.
2.  Click the top-left menu (☰) and select **Import from File**. Upload the `workflow.json` file from this repository.
3.  Go through each node that requires a credential (Google Drive, Twilio, Gemini) and select the credentials you just created from the dropdown menu.
4.  Set up a **Google Sheet** for logging with the headers: `Timestamp`, `User`, `Full Command`, `Status`, `Details`. Copy its **Sheet ID** from the URL and paste it into the `Google Sheets` node at the end of the workflow.

### 5. Connect Twilio to n8n

1.  In a terminal, run `ngrok http 5678` and copy the public `https://` URL.
2.  In your n8n `Webhook` node, copy the **Production URL path**.
3.  Combine them to get your full URL (e.g., `https://your-ngrok-url.io/webhook/your-path`).
4.  In your Twilio WhatsApp Sandbox settings, paste this full URL into the **"WHEN A MESSAGE COMES IN"** field and set the method to `HTTP POST`.
5.  **Activate** the workflow in n8n. You're all set!

### 💡 Technical Highlights & Problem-Solving
During development, several challenges related to the self-hosted n8n environment were encountered and overcome:
-   **Unreliable Look-Back Expressions**: Early versions of the workflow relied on "look-back" expressions (`$(...)`) to get data from previous nodes. This proved to be unreliable, especially in complex branches, leading to `undefined` errors. The architecture was refactored to use a more robust **"pass-it-forward" pattern** with `Set` and `Merge` nodes, ensuring data integrity throughout the workflow.
-   **Missing Core Nodes**: The initial Docker environment was running a faulty or extremely old version of n8n that was missing key nodes like `Read PDF`. After extensive debugging, a **full environment reset** (deleting the Docker volume and image) was performed to ensure a clean, modern n8n instance was running.
-   **File Content Extraction**: Due to the initial environment issues, a flexible approach to text extraction was developed. The final workflow uses n8n's built-in `Extract from File` node for PDFs and TXT files, and gracefully falls back to an external service (**ConvertAPI**) via an `HTTP Request` node for DOCX files, showcasing a hybrid and resilient design.

## Citations

This project utilizes several external services and APIs:
-   **n8n**: The core workflow automation platform.
-   **Twilio API**: For WhatsApp messaging.
-   **Google Drive & Google Sheets APIs**: For file management and logging.
-   **Google Gemini API**: For AI-powered text summarization.
-   **ConvertAPI**: As a fallback for extracting text from `.docx` files.

## 📜 License

This project is licensed under the MIT License.
