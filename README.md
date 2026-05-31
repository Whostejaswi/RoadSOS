# 🚨 RoadSOS — Emergency Response System

> Offline-capable, globally applicable emergency response platform for road accident victims. Instantly locates the nearest trauma hospitals, ambulance services, police stations, pharmacies, blood banks, puncture shops, and towing assistance — for any location on earth.

Built for the **CoERS IIT Madras Road Safety Hackathon 2026** — Problem Statement 3: Emergency Response Optimization.

Teammates:
Tejaswi K - B.Tech AI&DS
Poornaa Shree Praveenraj - B.E CSE

---

## What RoadSOS Does

When a road accident occurs, every second counts. RoadSOS gives the victim or bystander immediate access to:

- **Nearest trauma-capable hospitals** — ranked by distance AND capability score, not just proximity
- **Ambulance services** — both government (108) and private operators
- **Police stations** and **fire stations**
- **Pharmacies** and **blood banks**
- **Puncture shops** and **towing/garage services**
- **Country-specific emergency numbers** — auto-detected for 200+ countries

The system works **offline after first sync** — making it viable in rural areas and highway stretches where internet is unavailable.

---

## Key Features

- **Offline-first architecture** — syncs regional data from OpenStreetMap once, stores in local MySQL, runs without internet at query time
- **Global applicability** — same codebase works for any coordinate on earth; tested across India, Japan, UK, Brazil, UAE, and Kenya
- **Capability scoring** — hospitals ranked by bed count, specialties, operator type, and emergency facilities — not just distance
- **Trauma filter** — automatically excludes eye hospitals, dental clinics, maternity homes, and psychiatric facilities from emergency results
- **ETA estimation** — road-factor distance correction (×1.3) with average urban ambulance speed
- **Smart fallbacks** — police/fire stations show emergency numbers when direct phone is unavailable; puncture shops fall back to nearest garage
- **Country-aware emergency numbers** — verified override table for 30+ countries with known dataset errors

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python, FastAPI |
| Database | MySQL |
| Map Data | OpenStreetMap via Overpass API |
| Distance Calculation | Haversine formula (no routing API) |
| Reverse Geocoding | `reverse_geocoder` (offline) |
| Emergency Numbers | Bundled JSON — 200+ countries |
| Frontend | Streamlit, Folium/Leaflet |
| Location Resolution | Geopy / Nominatim |

---

## Project Structure

```
IITM hackathon/
├── roadsos_backend.py        # Core backend — all logic here
├── emergency_numbers.json    # Bundled emergency numbers — 200+ countries
├── app.py                    # Streamlit homepage
├── styles.css                # Dark theme CSS
├── components/
│   └── header.py
├── pages/
│   ├── 1_Dashboard.py        # About / purpose page
│   ├── 2_Emergency_Services.py
│   ├── 3_Map_View.py
│   ├── 4_SOS.py              # Primary SOS trigger page
│   └── 5_Emergency_Numbers.py
└── services/
    └── api.py                # Backend wrapper for frontend
```

---

## Setup Instructions

### Prerequisites

- Python 3.10+
- MySQL 8.0+ running locally
- Internet connection for first regional sync

### 1. Clone the repository

```bash
git clone https://github.com/Whostejaswi/roadsos.git
cd roadsos
```

### 2. Install dependencies

```bash
pip install streamlit streamlit-folium folium mysql-connector-python \
            reverse_geocoder requests geopy fastapi uvicorn
```

### 3. Set up MySQL database

Open MySQL Workbench or MySQL shell and run:

```sql
CREATE DATABASE roadsos;
USE roadsos;

CREATE TABLE hospitals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    osm_id VARCHAR(50) UNIQUE,
    name VARCHAR(255),
    lat DOUBLE,
    lon DOUBLE,
    phone VARCHAR(100),
    emergency BOOLEAN DEFAULT FALSE,
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(10) DEFAULT 'IN',
    confidence ENUM('high', 'medium') DEFAULT 'medium',
    specialty VARCHAR(255),
    capability_score INT DEFAULT 0
);

CREATE TABLE nearby_services (
    id INT AUTO_INCREMENT PRIMARY KEY,
    osm_id VARCHAR(50) UNIQUE,
    name VARCHAR(255),
    service_type VARCHAR(50),
    lat DOUBLE,
    lon DOUBLE,
    phone VARCHAR(100),
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(10) DEFAULT 'IN'
);

CREATE TABLE regions_synced (
    id INT AUTO_INCREMENT PRIMARY KEY,
    lat DOUBLE,
    lon DOUBLE,
    radius_km INT,
    country_code VARCHAR(10),
    synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ambulance_services (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255),
    phone VARCHAR(100),
    type ENUM('government', 'private') DEFAULT 'private',
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(10) DEFAULT 'IN',
    lat DOUBLE,
    lon DOUBLE
);

-- Seed government ambulance data
INSERT INTO ambulance_services (name, phone, type, city, state, country, lat, lon) VALUES
('108 Government Ambulance',    '108',          'government', 'Coimbatore', 'Tamil Nadu', 'IN', 11.0168, 76.9558),
('112 Emergency Response',      '112',          'government', 'Coimbatore', 'Tamil Nadu', 'IN', 11.0168, 76.9558),
('Ziqitza Ambulance (Private)', '1800-313-1414','private',    'Coimbatore', 'Tamil Nadu', 'IN', 11.0168, 76.9558),
('Falck Ambulance',             '044-28282828', 'private',    'Coimbatore', 'Tamil Nadu', 'IN', 11.0168, 76.9558),
('Red Cross Ambulance',         '1800-180-1234','private',    'Coimbatore', 'Tamil Nadu', 'IN', 11.0168, 76.9558);
```

### 4. Configure database credentials

Open `roadsos_backend.py` and update:

```python
DB_CONFIG = {
    "host":     "localhost",
    "user":     "root",
    "password": "your_password",  # ← change this
    "database": "roadsos"
}
```

### 5. Run the app

```bash
cd "RoadSOS"
streamlit run app.py
```

The app opens at `http://localhost:8501`

On first use for any location, the system fetches and caches regional data from OpenStreetMap (takes 30–60 seconds). All subsequent queries run instantly from the local database.

---

## How Offline Works

```
ONLINE (first sync per region)        OFFLINE (all subsequent use)
──────────────────────────────        ────────────────────────────
Overpass API → hospitals table        Haversine query → MySQL
Overpass API → nearby_services        Emergency numbers → local JSON
Mark region as synced in MySQL        Country code → reverse_geocoder
```

The `regions_synced` table tracks which areas have been cached. When a new location is detected within 40km of an already-synced region, no internet fetch is triggered.

---

## Testing

Run the backend directly to test across multiple locations:

```bash
python roadsos_backend.py
```

This runs the built-in test suite across Coimbatore, Tokyo, London, Krishnagiri, Dubai, São Paulo, and Nairobi — verifying hospitals, services, emergency numbers, and offline fallback behavior for each.

---

## API (FastAPI)

Start the REST API separately if needed:

```bash
uvicorn main:app --reload --port 8000
```

Endpoints:

| Endpoint | Method | Description |
|---|---|---|
| `/sos?lat=11.0168&lon=76.9558` | GET | Full SOS response for coordinates |
| `/sync?lat=11.0168&lon=76.9558` | GET | Trigger regional sync |
| `/health` | GET | API health check |
| `/docs` | GET | Auto-generated Swagger UI |

---

## Evaluation Criteria Coverage

| Criterion | How RoadSOS addresses it |
|---|---|
| Reliability and data accuracy | Real OSM data, verified emergency numbers, trauma-capable filter |
| Number of contacts fetched | Hospitals, police, fire, pharmacy, blood bank, puncture shop, towing |
| Offline functionality | MySQL cache, local JSON, offline reverse geocoding |
| Innovation and additional features | Capability scoring, ETA, global applicability, smart fallbacks |
| Information integration across countries | Tested across 7 countries, auto country detection |


