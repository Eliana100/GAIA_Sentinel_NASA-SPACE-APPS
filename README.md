Getting Started
Prerequisites

Before starting, make sure you have the following installed:

Docker Desktop: To containerize and run the application.

Git: To clone the repository.

(Optional for direct development): Node.js (v18+) and npm/yarn, Python (v3.9+) and pip, VS Code.

Local Run

Clone the Repository:

git clone https://github.com/your-username/cumaru.git
cd cumaru


Create the .env file:
Copy the example environment variables and fill them in.

cp .env.example .env
# Open .env and add your values, for example MAPBOX_TOKEN, NASA_USER, etc.


Note: For this prototype, many NASA/Twilio/Firebase variables can remain as placeholders.

Build and Run with Docker Compose:

docker-compose up --build


This command will:

Build the Docker images for the frontend and backend.

Start the frontend service (React app) at http://localhost:80.

Start the backend service (FastAPI app) at http://localhost:8000.

Initialize the SQLite database (data/cumaru.db).

Access the Application:
Open your web browser and navigate to http://localhost:80.

4. Environment Variables

The following environment variables are used. You will need to create a .env file in the root directory (and potentially backend/.env) with these values.

Variable Name	Description	Default (Prototype)
MAPBOX_TOKEN	Your Mapbox public access token (for Frontend)	pk.eyJ...
NASA_USER	NASA Earthdata login username	YOUR_NASA_USERNAME
NASA_PASS	NASA Earthdata login password	YOUR_NASA_PASSWORD
NASA_API_BASE_URL	Base URL for NASA Earthdata APIs (e.g., GPM, SMAP)	https://example.nasa.api
TWILIO_ACCOUNT_SID	Twilio Account SID (for SMS)	ACxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN	Twilio authentication token	your_auth_token
FIREBASE_API_KEY	Firebase API key (for Push Notifications)	AIz...
DATABASE_FILE	Path to the SQLite database file	data/cumaru.db
CORS_ORIGINS	Comma-separated list of allowed CORS origins	http://localhost:80,http://localhost:5173
5. API Endpoints

The FastAPI backend provides the following endpoints:

Base URL: http://localhost:8000

Tiles and Map Data

GET /api/tiles?bbox=...&zoom=...&time_range=...

Description: Returns map tiles with associated risk scores and the most recent resource data within a bounding box and specified zoom level. Triggers a simulated run of the alert engine for dynamic data.

Parameters:

bbox (string, required): Comma-separated string of coordinates west,south,east,north.

zoom (int, required): Map zoom level.

time_range (int, optional): Time range for data aggregation in hours (default: 24).

Example Response:

{
  "tiles": [
    {
      "tile_id": "tile_h3_8a12b0000000000",
      "h3_index": "h3_8a12b0000000000",
      "geom": "POINT(-46.63 -23.55)",
      "ts": "2023-10-27T10:30:00",
      "rain_24h": 65.2,
      "soil_moisture": 88.5,
      "score": 85,
      "type": "flood",
      "status": "active",
      "center_coords": {"lat": -23.55, "lon": -46.63}
    }
  ]
}

Alerts

GET /api/alerts/active?city=...&alert_type=...

Description: Lists all active alerts, with optional filtering by city and alert type. Triggers a simulated run of the alert engine.

Parameters:

city (string, optional): Filter by city name.

`alert

7. Alert Engine Thresholds and Rules

Here are example heuristic-based rules and thresholds used in the Alert Engine:

ALERT_THRESHOLD: 75 (Overall risk score required to trigger an alert).

CONFIDENCE_THRESHOLD: 0.6 (Minimum confidence level for an alert).

Flood Alert:

if rain_24h >= 50mm AND soil_moisture >= 75% => score += 30

if NDVI < 0.2 (low vegetation) => score += 10

Landslide Alert:

if slope > 20 degrees AND soil_moisture > 80% AND rain_48h > 70mm => score += 40

Prioritization:

final_score = base_score * (1 + (population_exposed / 100000)) (alerts in densely populated areas get higher priority).

Deduplication:

Alerts of the same type within 500 meters and 1 hour are grouped.

Rate limit: Maximum 1 alert / 10 min per user for push notifications.

Escalation: (Simulated) If an alert persists and confidence increases, trigger an escalation webhook.

8. Database Schema (SQLite: data/cumaru.db)
CREATE TABLE IF NOT EXISTS tiles (
    tile_id TEXT PRIMARY KEY,
    h3_index TEXT,
    geom TEXT,           -- Stores as GeoJSON string (e.g., POINT(-X -Y))
    meta_fields TEXT     -- Stores metadata as JSON string
);

CREATE TABLE IF NOT EXISTS tile_features (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tile_id TEXT NOT NULL,
    ts DATETIME NOT NULL,
    rain_1h REAL,
    rain_24h REAL,
    soil_moisture REAL,
    ndvi REAL,
    slope REAL,
    FOREIGN KEY (tile_id) REFERENCES tiles(tile_id)
);

CREATE TABLE IF NOT EXISTS alerts (
    alert_id TEXT PRIMARY KEY,
    tile_id TEXT NOT NULL,
    type TEXT NOT NULL,        -- e.g., 'flood', 'landslide', 'wildfire'
    score INTEGER NOT NULL,
    confidence REAL NOT NULL,
    status TEXT NOT NULL,      -- 'pending', 'active', 'resolved', 'false_positive'
    created_at DATETIME NOT NULL,
    resolved_at DATETIME,
    payload TEXT,              -- Stores AlertPayload JSON as string
    FOREIGN KEY (tile_id) REFERENCES tiles(tile_id)
);

CREATE TABLE IF NOT EXISTS crowd_reports (
    report_id TEXT PRIMARY KEY,
    user_id TEXT,
    lat REAL,
    lon REAL,
    type TEXT,                 -- e.g., 'flood', 'landslide', 'debris'
    photo_url TEXT,
    ts DATETIME,
    validated BOOLEAN          -- True if validated by civil defense
);

CREATE TABLE IF NOT EXISTS users (
    user_id TEXT PRIMARY KEY,
    phone TEXT,
    email TEXT,
    prefs TEXT,                -- JSON string for user preferences (e.g., monitored areas)
    location_geom TEXT         -- GeoJSON string for the user's primary location
);

CREATE TABLE IF NOT EXISTS organizations (
    org_id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    webhook_url TEXT           -- URL to receive alert notifications
);

9. Figma Mockups

These images represent the central design of the CUMARU dashboard at different zoom levels. You can find the high-quality PNGs in the .figma/mockups directory.

Global View (Central 3D globe with colored access points)
<img width="1024" height="1024" alt="Generated Image October 05, 2025 - 4_57PM (2)" src="https://github.com/user-attachments/assets/cca1c3b3-b6f2-4cff-8bc6-2a1e38aa1a1e" />

Country view (Brazil) (2D flat map with heat map divided by states/regions)
<img width="1024" height="1024" alt="Generated Image October 05, 2025 - 4_57PM (3)" src="https://github.com/user-attachments/assets/449935ba-dc3c-4f11-81d4-eda4f93834dc" />

State/Municipal View (São Paulo) (View closer to the map with alert pins and highlighted high-risk areas)
<img width="1024" height="1024" alt="Generated Image October 05, 2025 - 4_57PM (1)" src="https://github.com/user-attachments/assets/07851768-1dd5-428e-9a4b-fbb534a94d59" />

Neighborhood/zone view (example in São Paulo) (Ultra-detailed map with interactive layers)
<img width="1024" height="1024" alt="Generated Image October 05, 2025 - 4_57PM" src="https://github.com/user-attachments/assets/ba71417d-efb1-4964-845c-6a11787dcd63" />
