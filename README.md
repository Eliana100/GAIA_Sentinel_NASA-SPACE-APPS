# CUMARU - Urban Risk Monitoring System

The CUMARU project is a full-stack prototype demonstrating a comprehensive urban risk monitoring and alert system (for events like floods and landslides). It utilizes Earth observation data with a **demonstration of integration with real NASA APIs**. It offers an interactive frontend with a map, a robust backend with an API, and an alert engine, all orchestrated via Docker.

## Architecture Overview

The system is divided into layers:

1.  **Data Sources (NASA Integration / Simulated)**: The backend includes a `nasa_integrator` layer that demonstrates how to fetch data from real NASA APIs (GPM, MODIS, SMAP, SRTM). For the prototype, the processing of this raw data into `tile_features` is simulated via `data_simulator.py`.
2.  **Ingest Layer (Demonstrative)**: The `fetch_nasa_data_real` function in the backend simulates the call and consumption of data from NASA APIs.
3.  **Processing Layer (Alert Engine)**: The alert engine in the backend processes `tile_features` (whether simulated or derived from NASA data), applies heuristic rules, and generates alerts.
4.  **Feature Store / DB**: SQLite (`data/cumaru.db`) for persisting tiles, features, and alerts.
5.  **API Layer**: FastAPI backend exposing endpoints for the frontend and other services.
6.  **Dashboard (Frontend)**: React application with Mapbox GL JS for map and alert visualization.
7.  **Notification Layer (Simulated)**: Webhook endpoints for municipalities.

## Technologies Used

*   **Frontend**: React (Vite), Mapbox GL JS, Tailwind CSS
*   **Backend**: Python (FastAPI), SQLite, Pydantic, SQLAlchemy, `httpx` (for external calls)
*   **Orchestration**: Docker, Docker Compose
*   **Design**: Figma (mockups and JSON structure)

## Requirements

*   Docker and Docker Compose
*   Node.js and npm (for frontend development outside Docker, optional)
*   Python and pip (for backend development outside Docker, optional)

## Local Execution (with Docker)

1.  **Clone the Repository:**
    ```bash
    git clone https://github.com/your-username/cumaru.git
    cd cumaru
    ```

2.  **Configure Environment Variables:**
    Create `.env` files in `frontend/` and `backend/` based on the provided examples:

    `frontend/.env` (or `frontend/.env.local` for Vite):
    ```
    VITE_MAPBOX_TOKEN=YOUR_MAPBOX_ACCESS_TOKEN  # Generate a token at mapbox.com
    VITE_BACKEND_URL=http://localhost:8000
    ```
    `backend/.env`:
    ```
    DATABASE_URL=sqlite:///./data/cumaru.db
    NASA_EARTHDATA_USERNAME=YOUR_EARTHDATA_USERNAME # Create one at urs.earthdata.nasa.gov
    NASA_EARTHDATA_PASSWORD=YOUR_EARTHDATA_PASSWORD # Required to access some NASA APIs
    TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxx # Placeholder
    TWILIO_AUTH_TOKEN=your_auth_token # Placeholder
    TWILIO_PHONE_NUMBER=+15017122661 # Placeholder
    ```

    **Important**:
    *   Replace `YOUR_MAPBOX_ACCESS_TOKEN` with your actual Mapbox token.
    *   For real NASA integration, you'll need an Earthdata Login account (`NASA_EARTHDATA_USERNAME` and `NASA_EARTHDATA_PASSWORD`). The backend is already configured to include these authentication headers in the simulated call.

3.  **Build and Start Docker Containers:**
    ```bash
    docker-compose up --build -d
    ```
    This will build the frontend and backend images, create the SQLite database, and start the services.

4.  **Initialize Data (Optional, but Recommended):**
    To populate the database with simulated data and generate initial alerts, execute the scripts inside the backend container:
    ```bash
    docker-compose exec backend python -c "from backend.database import init_db; init_db()"
    docker-compose exec backend python backend/data_simulator.py
    docker-compose exec backend python backend/alert_engine.py --run-once
    ```
    The `alert_engine.py` can be set up as a cron job in a production environment, but for the prototype, we run it manually to populate data.

5.  **Access the Application:**
    Open your browser and navigate to `http://localhost:5173` (or the port Vite uses, which can be checked in Docker logs).

## API Endpoints (FastAPI Backend)

The backend API is available at `http://localhost:8000`.

*   **GET /api/tiles?bbox=[min_lon,min_lat,max_lon,max_lat]&zoom=[zoom_level]**
    *   Returns a list of tiles with `risk_score` and metadata within the specified bounding box and zoom level.
    *   Example return payload:
        ```json
        [
          {
            "tile_id": "h3_8a12b...",
            "lat": -23.55,
            "lon": -46.63,
            "risk_score": 75,
            "type": "flood",
            "timestamp": "2023-10-27T10:00:00Z",
            "popup_info": "High Risk: Flood in the central region. 24h Rain: 60mm."
          }
        ]
        ```

*   **GET /api/alerts/active**
    *   Returns a list of active alerts in the system.
    *   Example return payload (details in `alert_payload_examples.json`):
        ```json
        [
          {
            "alert_id": "A20251005-0001",
            "tile_id": "h3_8a12b...",
            "type": "flood",
            "score": 82,
            "confidence": 0.72,
            "status": "active",
            "created_at": "2025-10-05T14:30:00Z",
            "location_info": {
              "city": "São Paulo",
              "tile_description": "North Zone"
            },
            "recommendations": ["Evacuate riverside areas", "Avoid basements"],
            "evidence": {
              "rain_24h": 62,
              "soil_moisture": 86,
              "ndvi": 0.12,
              "chart_data": {
                "rain_48h": [
                  { "hour": "-48h", "value": 10 },
                  { "hour": "-24h", "value": 20 },
                  { "hour": "now", "value": 62 }
                ]
              }
            }
          }
        ]
        ```

*   **POST /api/ingest/nasa-data?start=...&end=...&product=gpm**
    *   **Real NASA Integration Demonstration**: This endpoint simulates calling a real NASA API to fetch Earth observation data.
    *   **Authentication**: Uses `NASA_EARTHDATA_USERNAME` and `NASA_EARTHDATA_PASSWORD` environment variables to simulate authentication.
    *   **Parameters**: `start` (ISO format date-time), `end` (ISO format date-time), `product` (gpm, modis, smap, srtm - to simulate different endpoints).
    *   For the prototype, the response is still simulated data, but the request structure for NASA is realistic.
    *   Example curl:
        ```bash
        curl -X POST "http://localhost:8000/api/ingest/nasa-data?start=2023-10-26T00:00:00Z&end=2023-10-27T00:00:00Z&product=gpm" -H "Authorization: Basic $(echo -n 'YOUR_EARTHDATA_USERNAME:YOUR_EARTHDATA_PASSWORD' | base64)"
        ```
    *   Returns: `{"message": "NASA data (simulated for prototype) processed for: gpm", "data": {...}}`

*   **POST /api/ingest/crowd-report**
    *   Accepts crowd-sourced reports (simulated).
    *   Method: `POST`
    *   JSON Payload:
        ```json
        {
          "user_id": "user123",
          "lat": -23.5505,
          "lon": -46.6333,
          "type": "flood",
          "photo_url": "http://example.com/photo.jpg",
          "description": "Street flooded near the station."
        }
        ```
    *   Returns: `{"message": "Crowd report received.", "report_id": "..."}`

*   **POST /api/webhook/municipality**
    *   Endpoint to receive acknowledgment/response webhooks from municipalities.
    *   Method: `POST`
    *   JSON Payload (example):
        ```json
        {
          "alert_id": "A20251005-0001",
          "status": "acknowledged",
          "message": "São Paulo Civil Defense acknowledged and taking action."
        }
        ```
    *   Returns: `{"message": "Municipal webhook received and processed."}`

## Data Simulation

The `scripts/generate_synthetic_data.py` script generates synthetic time series data for rain, soil moisture, and NDVI for tiles in example cities (São Paulo, Dhaka, Medellín). This data is used by `data_simulator.py` in the backend to populate the database with `tile_features`, which the `alert_engine` then processes. This step is crucial for the prototype's functionality, **even with the demonstrative NASA integration**, as the transformation from raw NASA data to `tile_features` format is complex and outside the scope of this initial prototype.

To generate synthetic data:
```bash
python scripts/generate_synthetic_data.py
