# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

This is a MERN (MongoDB, Express, React, Node.js) stack application deployed via Docker Compose with nginx as a reverse proxy. The architecture consists of four containerized services:

- **MongoDB** (mongo:7-jammy): Database layer with persistent volume storage
- **Backend API** (Node.js/Express): RESTful API server on port 5000
- **Frontend** (React/Vite): Static site built and served via nginx on port 80
- **Nginx**: Reverse proxy handling routing between frontend and backend

### Service Communication

- Nginx reverse proxy listens on port 80 (host)
- Frontend requests to `/api/*` are proxied to `backend:5000/api/*`
- All other requests go to the frontend container
- Backend connects to MongoDB at `mongodb://mongo:27017/merndb`
- All services communicate via the `mern-network` bridge network

### Data Model

The application uses a simple Mongoose schema defined in `backend/server.js:19-23`:
- `Item` model with fields: `name` (String), `description` (String), `createdAt` (Date)
- All items are sorted by `createdAt` descending in queries

### Frontend Architecture

- Single-page application using vanilla JavaScript (no React framework despite package.json)
- All code is in `frontend/index.html` - styles, markup, and JavaScript bundled together
- API URL resolution: uses `http://localhost:5000/api` for local development, `/api` for production
- Built with Vite and served from nginx in production

## Development Commands

### Docker Environment

**Start all services:**
```bash
docker-compose up -d
```

**View logs:**
```bash
docker-compose logs -f [service_name]  # service_name: mongo, backend, frontend, nginx
```

**Rebuild specific service:**
```bash
docker-compose up -d --build [service_name]
```

**Stop all services:**
```bash
docker-compose down
```

**Stop and remove volumes (clears database):**
```bash
docker-compose down -v
```

### Backend Development

**Navigate to backend:**
```bash
cd backend
```

**Install dependencies:**
```bash
npm install
```

**Run locally (requires MongoDB running):**
```bash
npm start  # Starts server on port 5000
```

**Environment variables:**
- `NODE_ENV`: Set to "production" in Docker
- `PORT`: Backend port (default: 5000)
- `MONGODB_URI`: MongoDB connection string

### Frontend Development

**Navigate to frontend:**
```bash
cd frontend
```

**Install dependencies:**
```bash
npm install
```

**Run development server:**
```bash
npm run dev  # Starts Vite dev server on port 3000
```

**Build for production:**
```bash
npm run build  # Outputs to dist/
```

**Preview production build:**
```bash
npm run preview
```

## API Endpoints

All backend routes are defined in `backend/server.js:27-62`:

- `GET /api/health` - Health check endpoint
- `GET /api/items` - Retrieve all items (sorted by createdAt descending)
- `POST /api/items` - Create new item (expects JSON body with name, description)
- `DELETE /api/items/:id` - Delete item by MongoDB ObjectId

## Key Configuration Files

- `docker-compose.yml`: Defines all services, networks, volumes, and dependencies
- `nginx/default.conf`: Nginx reverse proxy configuration with upstream definitions
- `backend/Dockerfile`: Single-stage production build using node:18-alpine
- `frontend/Dockerfile`: Multi-stage build (builder + nginx) for optimized production image
- `frontend/vite.config.js`: Vite configuration (dev server on port 3000)

## MongoDB Notes

- Database name: `merndb`
- Health check uses `mongosh` to ping the database
- Backend waits for healthy MongoDB before starting (`depends_on` with health condition)
- Data persists in Docker volume `mongo_data`
