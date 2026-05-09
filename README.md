# AWS Facial Recognition — ClassCheck

A Flask-based REST API that performs **facial recognition** using **AWS Rekognition**, comparing captured photos against reference student faces stored in **Amazon S3**. When a match is found, student attendance is automatically registered in **Amazon DynamoDB**. Includes a web-based camera interface for real-time photo capture.

This project was developed as a college study on cloud-based facial recognition, AWS services integration, and automated student attendance management.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Dependencies](#dependencies)
- [Environment Variables](#environment-variables)
- [Getting Started](#getting-started)
- [API Endpoints](#api-endpoints)
- [Web Interface](#web-interface)
- [Arduino Integration](#arduino-integration)
- [AWS Services Used](#aws-services-used)
- [DynamoDB Tables](#dynamodb-tables)
- [Recognition Flow](#recognition-flow)
- [Related Projects](#related-projects)
- [Credits](#credits)

---

## How It Works

1. A **photo is captured** — either through the web interface (browser camera), the Arduino image capture device, or a direct API call.
2. The image is sent as `multipart/form-data` to the Flask API's `POST /photos` endpoint.
3. The API checks if there is a **class scheduled for today** by querying the DynamoDB `Calendario` table.
4. If a class is scheduled and within the attendance time window (class start time + 10 minutes), the API:
   - **Lists all reference faces** stored in the S3 bucket
   - Uses **AWS Rekognition `CompareFaces`** to compare the captured photo against each reference image
   - When a match is found (≥ 80% similarity), the student is identified by their `matricula` (registration number, extracted from the S3 object key)
5. The student's **attendance is recorded** as `presente: true` in their DynamoDB history.
6. The web interface displays the **result** side-by-side with the matched database photo.

---

## Architecture

```
┌─────────────────────┐     ┌─────────────────────┐
│   Web Interface     │     │   Arduino Device     │
│   (index.html)      │     │   (Python serial)    │
│   Browser Camera    │     │   arduinoComunication│
└──────────┬──────────┘     └──────────┬──────────┘
           │  POST /photos (image)     │
           └──────────┬────────────────┘
                      ▼
            ┌──────────────────┐
            │   Flask API      │
            │   faceComparison │
            │   FlaskApi.py    │
            └────────┬─────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
  ┌────────────┐ ┌─────────┐ ┌──────────────┐
  │  AWS S3    │ │  AWS    │ │  DynamoDB    │
  │  (student  │ │  Rekog- │ │  (Calendario │
  │   photos)  │ │  nition │ │   & Aluno)   │
  └────────────┘ └─────────┘ └──────────────┘
```

---

## Project Structure

```
study-facial-recognition-aws/
├── public/
│   ├── faceComparisonFlaskApi.py   # Flask API — face comparison & attendance logic
│   └── index.html                  # Web UI — camera capture & result display
├── arduinoComunication.py          # Alternative: Python serial reader for Arduino
├── .gitignore
└── README.md
```

---

## Dependencies

| Package | Purpose |
|---------|---------|
| [Flask](https://palletsprojects.com/p/flask/) | Web framework for the REST API |
| [flask-cors](https://flask-cors.readthedocs.io/) | Cross-Origin Resource Sharing support |
| [boto3](https://boto3.amazonaws.com/) | AWS SDK for Python (S3, Rekognition, DynamoDB) |
| [python-dotenv](https://pypi.org/project/python-dotenv/) | Loads environment variables from `.env` file |
| [pytz](https://pypi.org/project/pytz/) | Timezone handling (`America/Sao_Paulo`) |

**Arduino communication only:**

| Package | Purpose |
|---------|---------|
| [pyserial](https://pypi.org/project/pyserial/) | Serial port communication with Arduino |
| [requests](https://docs.python-requests.org/) | HTTP client for sending images to the API |
| [opencv-python](https://pypi.org/project/opencv-python/) | Image processing (`cv2`, `numpy`) |

---

## Environment Variables

Create a `.env` file in the project root (it is already gitignored):

```env
AWS_ACCESS_KEY_ID=your-access-key-id
AWS_SECRET_ACCESS_KEY=your-secret-access-key
BUCKET=your-s3-bucket-name
TABLE_NAME_ALUNO=Aluno
TABLE_NAME_CALENDARIO=Calendario
```

| Variable | Description |
|----------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS access key with S3, Rekognition, and DynamoDB permissions |
| `AWS_SECRET_ACCESS_KEY` | AWS secret access key |
| `BUCKET` | S3 bucket name containing student reference photos |
| `TABLE_NAME_ALUNO` | DynamoDB table name for student records |
| `TABLE_NAME_CALENDARIO` | DynamoDB table name for class schedule/calendar |

---

## Getting Started

### Prerequisites

- **Python 3.8+**
- An **AWS account** with the following services enabled:
  - **S3** — for storing student reference photos
  - **Rekognition** — for facial comparison
  - **DynamoDB** — for student and schedule data
- A webcam (for browser-based capture)

### Local Setup

1. **Clone the repository:**
   ```bash
   git clone git@github.com:WagnerCaetano/study-facial-recognition-aws.git
   cd study-facial-recognition-aws
   ```

2. **Create a virtual environment and install dependencies:**
   ```bash
   python -m venv venv
   source venv/bin/activate   # Linux/Mac
   venv\Scripts\activate      # Windows

   pip install flask flask-cors boto3 python-dotenv pytz
   ```

3. **Create the `.env` file** with your AWS credentials and resource names (see [Environment Variables](#environment-variables)).

4. **Set up S3 bucket:**
   - Create an S3 bucket in the `sa-east-1` region
   - Upload student reference photos using the path pattern: `public/{matricula}.jpg`
   - Example: `public/2023001.jpg` where `2023001` is the student's registration number

5. **Set up DynamoDB tables** (see [DynamoDB Tables](#dynamodb-tables)).

6. **Run the Flask API:**
   ```bash
   cd public
   python faceComparisonFlaskApi.py
   ```

7. **Open the web interface** at `http://localhost:5000`

---

## API Endpoints

### `GET /`

Serves the web interface (`index.html`) with camera capture functionality.

---

### `POST /photos`

Receives a photo and performs facial recognition against the S3 reference images.

**Request:**
- Content-Type: `multipart/form-data`
- Body: `file` — the captured image (PNG/JPG)

**Response codes:**

| Status | Message | Description |
|--------|---------|-------------|
| `201` | `Presente: {matricula}` | Student recognized and attendance registered |
| `200` | `Ja esta presente` | Student already marked present today |
| `400` | `Nao foi possivel identificar` | No matching face found in S3 |
| `404` | `Nao ha aulas hoje` | No classes scheduled for today |
| `409` | `Ja foi o tempo de marcar presença` | Outside the attendance time window |
| `500` | Error message | Internal server error |

**Example response (match found):**
```json
{
  "message": "Presente:",
  "matricula": "2023001",
  "status": 201
}
```

**Example response (no match):**
```json
{
  "message": "Nao foi possivel identificar",
  "status": 400
}
```

---

## Web Interface

The `index.html` provides a browser-based interface with:

- **Live camera feed** — uses `navigator.mediaDevices.getUserMedia()` to access the webcam
- **Capture button** — takes a screenshot from the video stream
- **Side-by-side comparison** — displays the captured image alongside the matched database photo from S3
- **Result display** — shows the recognition result (student ID or error message)

The captured image is sent to the `/photos` endpoint via `fetch()` with `FormData`. The API URL is configured in the `sendImage()` JavaScript function.

---

## Arduino Integration

The `arduinoComunication.py` script provides an alternative to the browser-based capture. It reads image data from an Arduino connected via serial port and forwards it to the API.

**Configuration:**
- Serial port: `COM5` at `2000000` baud rate (modify in the script to match your setup)
- API URL: configured in the `url` variable

**Usage:**
```bash
pip install pyserial requests opencv-python numpy
python arduinoComunication.py
```

Press `Ctrl+C` to stop the script. This is the Python equivalent of the Java-based [study-imagine-capture-arduino](https://github.com/WagnerCaetano/study-imagine-capture-arduino) project.

---

## AWS Services Used

| Service | Region | Purpose |
|---------|--------|---------|
| **Amazon S3** | `sa-east-1` | Stores student reference photos (`public/{matricula}.jpg`) |
| **Amazon Rekognition** | `us-east-1` | `CompareFaces` API for facial matching (≥ 80% similarity threshold) |
| **Amazon DynamoDB** | `sa-east-1` | Student records (`Aluno`) and class schedule (`Calendario`) |

---

## DynamoDB Tables

### Calendario (Calendar)

| Key | Type | Description |
|-----|------|-------------|
| `lista-dias-aulas` (PK) | String | Day of the week in Portuguese (`segunda`, `terça`, `quarta`, `quinta`, `sexta`, `sábado`, `domingo`) |
| `horario` | String | Class start time in `HH:MM` format |

### Aluno (Student)

| Key | Type | Description |
|-----|------|-------------|
| `matricula` (PK) | String | Student registration/ID number |
| `historico` | List | Array of attendance records |
| `historico[].data` | String | Date in `DD/MM/YYYY` format |
| `historico[].dia` | String | Day of the week in Portuguese |
| `historico[].hora` | String | Time in `HH:MM` format |
| `historico[].id_historico` | String | Sequential history entry ID |
| `historico[].presente` | Boolean | Whether the student was present |

---

## Recognition Flow

```
Photo captured (browser / Arduino / API)
         │
         ▼
  Check class schedule ──── No class today ──▶ 404
         │
     Class today
         │
         ▼
  Check time window ──── Outside window ──▶ 409
         │
   Within class time
   (start to start+10min)
         │
         ▼
  List S3 reference images
         │
         ▼
  For each image:
    Rekognition CompareFaces
    (threshold: 80%)
         │
    ┌────┴────┐
    │         │
  Match    No match
    │         │
    ▼         ▼
  Extract   Try next image
  matricula    │
    │       All exhausted
    ▼          │
  Check if   ▼
  already   400 "Not identified"
  present
    │
  ┌──┴──┐
  Yes   No
  │     │
  ▼     ▼
 200   201 "Present"
```

---

## Related Projects

These projects form the complete **ClassCheck** attendance system:

- **[study-imagine-capture-arduino](https://github.com/WagnerCaetano/study-imagine-capture-arduino)** — Arduino + Java desktop application that captures student images via serial communication and sends them to this API (alternative to the browser-based capture)
- **[study-lambda-step-function-AWS](https://github.com/WagnerCaetano/study-lambda-step-function-AWS)** — AWS Lambda + Step Functions that automatically marks students as absent if they were not recognized during class time

---

## Credits

- **Author:** [Wagner Caetano](https://github.com/WagnerCaetano)
- Developed as an academic study on cloud-based facial recognition (AWS Rekognition), serverless architectures, and automated student attendance management.

---

## License

This project is intended for educational and academic purposes.