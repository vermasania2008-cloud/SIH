# AIRIS - Airfare Index for India

AIRIS is a defensible, reproducible, auditable Indian domestic airfare price-index system that measures like-for-like fare movement. Designed as a production-oriented aviation intelligence and airfare analytics platform.

## Features

- **National Airfare Index:** Real-time monitoring and analysis of airfare movement across India's aviation network.
- **Route Intelligence & Airport Analytics:** Detailed analysis at route and airport levels.
- **Enterprise-Grade Dashboard:** A highly professional, data-centric interface for analysts and policymakers using Recharts for interactive visualizations.
- **Data Engineering Pipelines:** Robust processing for raw quotes into validated observations and aggregations.
- **Lineage Tracing:** Full auditability from final index value down to contributing cell prices.

## Architecture

- **Backend / API**: FastAPI (Python) serving a RESTful API with Pydantic for validation and SQLAlchemy for ORM.
- **Database**: PostgreSQL (Supabase) for storing raw quotes, processed observations, and index values.
- **Frontend Dashboard**: React + Vite application tailored with a custom enterprise design system (Recharts, Lucide Icons, modular UI architecture).
- **Data Processing**: Pandas and NumPy for index calculation estimators (GEKS-Jevons, TPD, Naive-Mean).
- **Packaging**: Docker & Docker Compose for reproducible builds, alongside Poetry for Python dependency management.

## Setup Instructions

### Prerequisites
- Node.js (v18+)
- Python (3.11+)
- Poetry (for Python dependency management)

### 1. Backend (API)
1. Navigate to the project root.
2. Copy the environment variables template: `cp .env.example .env`
3. Update `.env` with your **Supabase Database URL** and **API Keys**.
4. Install dependencies via Poetry: `poetry install` (or install manually from `requirements.txt`)
5. Run migrations to initialize the database: `poetry run alembic upgrade head`
6. Run the API server: 
   ```bash
   poetry run uvicorn app.main:app --reload
   ```
   *The API will be available at `http://localhost:8000`*

### 2. Frontend (React Dashboard)
1. Navigate to the frontend directory: `cd frontend`
2. Install dependencies:
   ```bash
   npm install
   ```
3. Run the development server:
   ```bash
   npm run dev
   ```
   *The dashboard will be available at `http://localhost:5173`*

### 3. Docker (Optional)
Run the entire stack using Docker Compose:
```bash
docker-compose up -d
```
