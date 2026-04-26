# Aqua Trajectory AI

A full-stack application with Python FastAPI backend and Node.js frontend.

## Quick Start

### Prerequisites
- Docker
- Docker Compose

### Run with Docker Compose

```bash
docker-compose up --build
```

This will:
- Build and start the backend service on `http://localhost:8000`
- Build and start the frontend service on `http://localhost:3000`

### Access the Application

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:8000
- **Backend Health Check**: http://localhost:8000/api/health
- **Frontend Health Check**: http://localhost:3000/api/health

### Stopping the Application

```bash
docker-compose down
```

## Project Structure

```
aqua-trajectory-ai/
├── backend/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── docker-compose.yml
└── README.md
```