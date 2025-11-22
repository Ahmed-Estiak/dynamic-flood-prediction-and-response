# Flood Intelligence System – High-Level Architecture

## 1. Overview  
This project is a cloud-deployed Flood Risk Prediction & Response System for Bangladesh that combines machine learning, geospatial data, and automation to estimate flood risk, visualise it on a map, and trigger situational alerts. The system has four main parts:  
- Backend ML API (Python + FastAPI, later wrapped in Azure Functions)  
- Frontend map UI (React + Azure Maps)  
- Automation / agent layer (n8n)  
- Data & analytics layer (water-point, population, displacement modelling)  

---

## 2. Backend Runtime  
- Language: Python  
- Framework: FastAPI (in development locally), to be deployed as an HTTP-triggered Azure Function.  
- Responsibilities:  
  - Load trained flood-risk model and configuration (initially from local files, later from Azure Blob storage)  
  - Expose `/predict` endpoint for:  
    - **Per-point prediction** (lat/lon input or full feature vector)  
    - **Per-cell prediction** (grid cell_id input)  
  - Future endpoints for: `/population_scenario`, `/waterpoint_risk`, `/checkin` (user check-in)  

---

## 3. Data Storage  
**For now (development):**  
- Model: `backend/model/flood_model.pkl`  
- Config: `backend/model/model_config.json`  
- Real and semi-real data layers:  
  - `data/grid.geojson` (grid cells + features derived from real datasets — elevation (Copernicus GLO-30), rainfall (CHIRPS or national), river distance, etc.)  
  - `data/baseline_population.geojson` (population per grid cell derived from real gridded population data – e.g., WorldPop for Bangladesh)  
  - `data/mwater_sites.geojson` (water-point layer: real if accessible via API/export; otherwise realistic synthetic layer based on actual settlement locations)  
**Later (cloud):**  
- Azure Blob Storage for model, config, GeoJSON layers  
- Optional: Azure Table / Azure SQL for logs, check-ins  

---

## 4. Geospatial Design  
- Use a fixed grid or polygon cell system stored as **GeoJSON** covering Bangladesh.  
- Each grid cell contains:  
  - `cell_id`  
  - `rainfall_mm` (aggregated from real rainfall dataset)  
  - `soil_moisture` (derived or approximated)  
  - `distance_to_river_km`  
  - `elevation_m` (from Copernicus GLO-30 DEM)  
  - `historical_flood_frequency` (derived from global flood-hazard data)  
- The model predicts for each cell (and optionally for a point):  
  - `flood_probability` (0-1)  
  - `risk_category` (low, medium, high)  
Results are visualised as heatmap layers in the frontend.

---

## 5. Frontend Maps  
- Tech: React + Azure Maps Web SDK  
- Features:  
  - Base map showing flood-risk heatmap (GeoJSON layer)  
  - On-click popup showing: flood probability, risk category, key feature values for that location  
  - Additional layers:  
    - Water points (colour coded by risk)  
    - Baseline vs displaced population density layers  

---

## 6. Automation (n8n)  
- n8n runs locally in Docker during development.  
- Core workflow (later in production):  
  - Trigger: Cron (e.g. hourly)  
  - Fetch live environmental data (rainfall updates, river level proxies)  
  - Call backend `/predict` (for key locations or grid cells)  
  - Aggregate high-risk areas  
  - Use LLM node to generate human-readable alerts  
  - Send alerts via Telegram or email  
- Later, n8n may be deployed to Azure Container Apps or n8n Cloud.

---

## 7. Water-point, Population & Displacement  
- **Water-point integration**:  
  - GeoJSON `mwater_sites.geojson` containing location, type (borehole, handpump, etc.), population served (if available)  
  - If real mWater export unavailable, use synthetic layer using real settlement data  
- **Baseline population**:  
  - `baseline_population.geojson` derived from real gridded population data (e.g., WorldPop Bangladesh) aggregated to cells  
- **Displacement modelling** (Python, initially offline batch):  
  - Input: flood risk per cell + baseline population  
  - Output: remaining population per cell, displaced population per cell  
  - Exposed (in later version) via `/population_scenario` endpoint  
- **“I am here / I am safe” check-in**:  
  - Endpoint `/checkin` where voluntary users submit their status  
  - Data used only for R&D calibration, not real-time decisions  

---

## 8. Infrastructure as Code (Terraform)  
- Terraform project in `/infra` includes:  
  - Azure Resource Group  
  - Azure Storage Account (Blob)  
  - Azure Function App (backend)  
  - Azure Maps account  
  - (Optional) Static Web App or App Service for frontend  
- All cloud resources are managed via `terraform apply`, not manual setup  

---

## 9. Repos & Structure  
- `/backend` – FastAPI app, model loader, Azure Function wrapper  
- `/frontend` – React + Azure Maps UI  
- `/infra` – Terraform configuration  
- `/notebooks` – ML experiment notebooks, displacement modelling  
- `/data` – Local GeoJSON/CSV/TIFF used for development (DEM, population, rainfall, grid, etc.)  
- `/scripts` – Helper scripts (environment checks, data-prep, grid generation)  

---

This architecture uses **real open datasets for Bangladesh where possible** while maintaining flexibility and clarity about synthetic vs real data layers. It’s stable enough to build on now and extensible for future enhancements.
