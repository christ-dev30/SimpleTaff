# Simple Three (SimpleTaff) — Platform & Marketing Showcase

## 📌 Presentation
**Simple Three** is an enterprise-grade multi-tenant SaaS platform designed to digitize the complete lifecycle of placed workforce (security guards, industrial cleaning, construction, logistics) for staffing agencies and labor contractors.

---

## 📁 Repository Structure
```
simple-three/
├── backend/                   # Spring Boot 3.x / Java 21 REST API (Flyway, JPA, JWT, Security)
├── frontend-app/              # Multi-role SPA (Admin Entreprise, Coordonnateur, Employeur Site)
├── site-vitrine/              # Marketing showcase & lead generation portal
├── docs/                      # Technical specifications & Database schema
│   └── schema-bdd.md
└── docker-compose.yml         # Containerized orchestration (PostgreSQL + Spring Boot)
```

---

## 🚀 Quick Start Guide

### Prerequisites
- JDK 21+
- PostgreSQL 15+
- Maven 3.8+

### 1. Database Setup
Ensure PostgreSQL is running locally on port `5432` with a database named `simpletaff` (or launch via Docker):
```bash
docker-compose up -d postgres
```

### 2. Launch Backend API
```bash
cd backend
mvn clean spring-boot:run
```
The REST API will start at `http://localhost:8080`. Flyway automatically creates and seeds the database schemas.

### 3. Launch Frontend App & Marketing Site
Serve static assets using any standard Web server or directly open `frontend-app/index.html` and `site-vitrine/index.html` in your browser.

---

## 🛡️ User Roles & Demo Credentials

| Role | Email | Password | Access |
| :--- | :--- | :--- | :--- |
| **ADMIN_ENTREPRISE** | `admin@prosecurite.ci` | `admin123` | Full RH, Payroll, Billing, Audit |
| **COORDONNATEUR** | `coord@prosecurite.ci` | `coord123` | Enrollment, Missions, Equipment |
| **EMPLOYEUR / SITE** | `employeur@sitex.ci` | `emp123` | QR Scanner, Attendance Validation |

---

## 🧪 Acceptance Test Mapping (Plan de Recette)

1. **TC-01 (Multi-Tenant Isolation)**: Verified via `TenantContext` & Spring Security filter in `backend/src/main/java/com/siege/platform/config/TenantInterceptor.java`.
2. **TC-02 (File Upload Quota 13MB)**: Validated in `backend/src/main/java/com/siege/platform/agent/AgentController.java` & frontend form `validateFileSize()`.
3. **TC-03 (HMAC QR Code Signature)**: Validated in `backend/src/main/java/com/siege/platform/pointage/HmacQrService.java`.
4. **TC-04 (GPS Geofencing Anomaly)**: Calculated in `PointageService.java` using Haversine formula against `site.rayonGeofenceMetres`.
5. **TC-05 (Expiration Scheduler)**: Managed by `@Scheduled` method in `ExpirationScheduler.java` daily at midnight.
6. **TC-06 (Automated Payroll Calculation)**: Formulated in `PaieService.java` for prorated base, CNPS/CNAM contributions, and unexcused absence deductions.