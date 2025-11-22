# Flood Intelligence System – High-Level Architecture

## 1. Overview

This project is a cloud-deployed Flood Risk Prediction & Response System that combines machine learning, geospatial data, and automation to predict flood risk, visualize it on a map, and trigger alerts.

The system has four main parts:
- Backend ML API (Python + FastAPI, later wrapped in Azure Functions)
- Frontend map UI (React + Azure Maps)
- Automation / agent layer (n8n)
- Data and analytics layer (mWater, population, displacement model)

---

## 2. Backend Runtime

- Language: Python
- Framework: FastAPI (locally), deployed as an HTTP-triggered Azure Function
- Responsibilities:
  - Load trained flood risk model and config from local files (or Azure Blob later)
  - Expose `/predict` endpoint for:
    - Per-point prediction (lat/lon or feature vector)
    - Per-cell prediction (grid cell id)
  - Later: endpoints for population scenarios, water-point risk, etc.

---

## 3. Data Storage

**For now (development):**
- Model file: `backend/model/flood_model.pkl`
- Config: `backend/model/model_config.json`
- Sample data:
  - `data/grid.geojson` (grid cells and features)
  - `data/mwater_sites.geojson` (mock mWater water points)
  - `data/baseline_population.geojson` (population per grid cell)

**Later (cloud):**
- Azure Blob Storage for:
  - Model
  - Config
  - GeoJSON layers
- Optional: Azure Table/SQL for check-ins and logs.

---

## 4. Geospatial Design

- Use a fixed grid or polygons stored as **GeoJSON**.
- Each grid cell contains:
  - `cell_id`
  - `rainfall_mm`
  - `soil_moisture`
  - `distance_to_river_km`
  - `elevation_m`
  - `historical_flood_frequency`
- The model predicts:
  - `flood_probability` (0–1)
  - `risk_category` (low, medium, high)

These results are visualized as heatmap layers in the frontend.

---

## 5. Frontend Maps

- Tech: React + Azure Maps Web SDK.
- Features:
  - Base map with flood-risk heatmap (GeoJSON layer)
  - On-click popup:
    - Flood probability
    - Risk category
    - Key feature values
  - Optional layers:
    - mWater water points (colored by risk)
    - Baseline vs displaced population density

---

## 6. Automation (n8n)

- n8n runs as a local Docker container during development.
- Core workflow (later): 
  - Trigger: Cron (e.g. every hour)
  - Fetch live weather / river proxy data
  - Call backend `/predict` for key locations or cells
  - Aggregate high-risk areas
  - Use LLM node to generate human-readable alerts
  - Send alerts via Telegram or email

In the future, n8n can be moved to:
- Azure Container Apps
- Or n8n Cloud

---

## 7. mWater, Population, Displacement

- mWater integration (initially mock data):
  - `mwater_sites.geojson` with:
    - water point location
    - type
    - population served (approximate)
- Baseline population data:
  - `baseline_population.geojson` with:
    - `cell_id`
    - `population`
- Displacement model (Python, offline / batch at first):
  - Input: flood risk per cell + baseline population
  - Output:
    - `remaining_population` per cell
    - `displaced_population` per cell
  - Later exposed via `/population_scenario` API endpoint
- “I am here / I am safe”:
  - Simple `/checkin` endpoint
  - Stores voluntary user check-ins for **R&D calibration only**, not real-time decision-making.

---

## 8. Infrastructure as Code (Terraform)

- Terraform project in `/infra`:
  - Azure Resource Group
  - Storage Account
  - Azure Function App for backend
  - Azure Maps account
  - (Optional) Static Web App or App Service for frontend
- All cloud resources are created and updated via `terraform apply`, never by hand.

---

## 9. Repos and Structure

- `/backend` – FastAPI app, model loader, Azure Function wrapper
- `/frontend` – React + Azure Maps UI
- `/infra` – Terraform configuration
- `/notebooks` – ML experiments, displacement modeling
- `/data` – Local GeoJSON/CSV used for development
- `/scripts` – Helper scripts (env checks, data prep, etc.)

This architecture is stable enough to build on and flexible enough to extend.
