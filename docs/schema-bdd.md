# Architecture & Design Technical Specification: Simple Three (SimpleTaff)

## Executive Summary
This document outlines the complete architectural design and database model for **Simple Three** (SimpleTaff), a multi-tenant SaaS platform for workforce placement, field management, secure attendance tracking, automated payroll, client billing, and administrative alert management.

---

## 1. Directory & Repository Structure

```
simple-three/
├── backend/                                  # Spring Boot 3.x / Java 21 REST API
│   ├── src/main/java/com/simpletaff/
│   │   ├── SimpleTaffApplication.java
│   │   ├── config/                           # Security, WebMvc, CORS, Async Config
│   │   │   ├── SecurityConfig.java
│   │   │   ├── WebMvcConfig.java
│   │   │   └── TenantInterceptor.java
│   │   ├── security/                         # JWT, Tenant Context, Password Encoder
│   │   │   ├── JwtTokenProvider.java
│   │   │   ├── JwtAuthenticationFilter.java
│   │   │   ├── HmacQrService.java
│   │   │   └── TenantContext.java
│   │   ├── domain/                           # JPA Entities
│   │   │   ├── Entreprise.java
│   │   │   ├── Utilisateur.java
│   │   │   ├── AgentTerrain.java
│   │   │   ├── CarteAgent.java
│   │   │   ├── StructureDemandeuse.java
│   │   │   ├── Site.java
│   │   │   ├── Poste.java
│   │   │   ├── AffectationMetier.java
│   │   │   ├── Pointage.java
│   │   │   ├── BulletinDePaie.java
│   │   │   ├── Facture.java
│   │   │   ├── AuditLog.java
│   │   │   ├── DocumentAgent.java
│   │   │   ├── NotificationEvenement.java
│   │   │   └── ContactLead.java
│   │   ├── repository/                       # Spring Data Repositories
│   │   ├── service/                          # Business logic
│   │   │   ├── AuthService.java
│   │   │   ├── AgentService.java
│   │   │   ├── PointageService.java
│   │   │   ├── PaieService.java
│   │   │   ├── FactureService.java
│   │   │   ├── AuditService.java
│   │   │   ├── ExpirationScheduler.java
│   │   │   └── LeadService.java
│   │   └── web/rest/                         # REST Controllers
│   │       ├── AuthController.java
│   │       ├── AgentController.java
│   │       ├── PointageController.java
│   │       ├── PaieController.java
│   │       ├── FactureController.java
│   │       ├── AuditController.java
│   │       ├── LeadController.java
│   │       └── ReferentielController.java
│   ├── src/main/resources/
│   │   ├── application.yml
│   │   └── db/migration/                     # Flyway SQL Scripts
│   │       ├── V1__init_schema.sql
│   │       └── V2__seed_data.sql
│   └── pom.xml
├── frontend-app/                             # Multi-role SPA Frontend
│   ├── index.html                            # Master Login & Role Selector
│   ├── admin-entreprise/
│   │   └── index.html                        # Admin Dashboard (RH, Paie, Factures, Audit)
│   ├── coordonnateur/
│   │   └── index.html                        # Coordinator Dashboard (Enrollment, Missions, Pointages)
│   ├── employeur/
│   │   └── index.html                        # Employer Site Dashboard (Scanner, Validation, Reports)
│   ├── css/
│   │   └── main.css                          # Shared Glassmorphism Design Tokens & Utilities
│   └── js/
│       ├── app.js                            # Core Router, State & API Client
│       ├── auth.js                           # Auth state & JWT storage
│       └── utils.js                          # Geo distance, QR scanner helpers, File quota check
├── site-vitrine/                             # Independent Public Marketing Website
│   ├── index.html                            # Landing Page (Hero, Problem, Features, Pricing, FAQ, Contact)
│   ├── css/
│   │   └── style.css                         # Marketing Landing CSS (Emerald/Blue theme)
│   └── js/
│       └── landing.js                        # FAQ accordions, Lead submission form, Smooth scroll
├── docs/
│   ├── schema-bdd.md                         # Full ERD & Schema Spec
│   └── api-collection.json                   # OpenAPI / Postman Collection
└── docker-compose.yml                        # Postgres + Spring Boot + Nginx Orchestration
```

---

## 2. Database Schema (PostgreSQL Flyway Scripts)

```sql
-- Flyway V1__init_schema.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. Entreprise (Tenant)
CREATE TABLE entreprise (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    nom VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    statut VARCHAR(50) NOT NULL DEFAULT 'ACTIF',
    formule_abonnement VARCHAR(50) NOT NULL DEFAULT 'STANDARD',
    taux_cotisation NUMERIC(5, 2) NOT NULL DEFAULT 6.50,
    taux_retenue_reduite NUMERIC(5, 2) NOT NULL DEFAULT 5.00,
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 2. Utilisateur (RBAC)
CREATE TABLE utilisateur (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID REFERENCES entreprise(id) ON DELETE CASCADE,
    nom VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    mot_de_passe VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL, -- SUPER_ADMIN, ADMIN_ENTREPRISE, COORDONNATEUR, EMPLOYEUR, AGENT
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 3. Structure Demandeuse (Client Final)
CREATE TABLE structure_demandeuse (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    nom VARCHAR(255) NOT NULL,
    contact_nom VARCHAR(255),
    contact_email VARCHAR(255),
    contact_telephone VARCHAR(50),
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 4. Site
CREATE TABLE site (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    structure_demandeuse_id UUID NOT NULL REFERENCES structure_demandeuse(id) ON DELETE CASCADE,
    nom VARCHAR(255) NOT NULL,
    commune VARCHAR(100) NOT NULL,
    latitude NUMERIC(10, 8) NOT NULL,
    longitude NUMERIC(11, 8) NOT NULL,
    rayon_geofence_metres INT NOT NULL DEFAULT 500,
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 5. Poste
CREATE TABLE poste (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    site_id UUID NOT NULL REFERENCES site(id) ON DELETE CASCADE,
    intitule VARCHAR(255) NOT NULL,
    tarif_journalier NUMERIC(12, 2) NOT NULL,
    salaire_base_mensuel NUMERIC(12, 2) NOT NULL,
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 6. Agent Terrain
CREATE TABLE agent_terrain (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    nom VARCHAR(255) NOT NULL,
    prenom VARCHAR(255) NOT NULL,
    contact VARCHAR(50) NOT NULL,
    telephone_secondaire VARCHAR(50),
    photo_url VARCHAR(512),
    contact_urgence_nom VARCHAR(255) NOT NULL,
    contact_urgence_telephone VARCHAR(50) NOT NULL,
    contact_urgence_lien VARCHAR(100) NOT NULL,
    statut VARCHAR(50) NOT NULL DEFAULT 'EN_SERVICE',
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 7. Carte Agent (Badge QR/NFC)
CREATE TABLE carte_agent (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agent_id UUID UNIQUE NOT NULL REFERENCES agent_terrain(id) ON DELETE CASCADE,
    code_qr TEXT NOT NULL,
    identifiant_nfc VARCHAR(255),
    statut VARCHAR(50) NOT NULL DEFAULT 'ACTIF',
    mis_a_jour_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 8. Affectation Métier (Historisée)
CREATE TABLE affectation_metier (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL REFERENCES agent_terrain(id) ON DELETE CASCADE,
    poste_id UUID NOT NULL REFERENCES poste(id) ON DELETE CASCADE,
    date_debut DATE NOT NULL,
    date_fin DATE,
    statut VARCHAR(50) NOT NULL DEFAULT 'ACTIF',
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 9. Document Agent
CREATE TABLE document_agent (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agent_id UUID NOT NULL REFERENCES agent_terrain(id) ON DELETE CASCADE,
    type_document VARCHAR(100) NOT NULL,
    fichier_url VARCHAR(512) NOT NULL,
    taille_octets BIGINT NOT NULL,
    date_expiration DATE NOT NULL,
    statut VARCHAR(50) NOT NULL DEFAULT 'VALIDE',
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 10. Pointage
CREATE TABLE pointage (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    affectation_id UUID NOT NULL REFERENCES affectation_metier(id) ON DELETE CASCADE,
    carte_id UUID REFERENCES carte_agent(id),
    date_heure_entree TIMESTAMP WITH TIME ZONE NOT NULL,
    date_heure_sortie TIMESTAMP WITH TIME ZONE,
    mode VARCHAR(50) NOT NULL, -- QR_CODE, NFC, PHOTO_GPS, BIOMETRIE
    latitude_entree NUMERIC(10, 8),
    longitude_entree NUMERIC(11, 8),
    selfie_url VARCHAR(512),
    anomalie VARCHAR(255), -- Ex: ANOMALIE_GPS, RETARD, SORTIE_ANTICIPEE
    valide_par_employeur BOOLEAN DEFAULT TRUE,
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 11. Bulletin De Paie
CREATE TABLE bulletin_de_paie (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL REFERENCES agent_terrain(id) ON DELETE CASCADE,
    periode VARCHAR(7) NOT NULL, -- YYYY-MM
    jours_prevus INT NOT NULL,
    jours_valides INT NOT NULL,
    jours_absence INT NOT NULL,
    salaire_base_proratise NUMERIC(12, 2) NOT NULL,
    total_primes NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    cotisation_cnps NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    cotisation_cnam NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    retenues_absences NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    salaire_net NUMERIC(12, 2) NOT NULL,
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 12. Facture
CREATE TABLE facture (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID NOT NULL REFERENCES entreprise(id) ON DELETE CASCADE,
    structure_demandeuse_id UUID NOT NULL REFERENCES structure_demandeuse(id) ON DELETE CASCADE,
    periode VARCHAR(7) NOT NULL,
    montant_total NUMERIC(12, 2) NOT NULL,
    statut VARCHAR(50) NOT NULL DEFAULT 'EN_ATTENTE', -- EN_ATTENTE, PAYEE, EN_RETARD
    rapport_pointage_url VARCHAR(512),
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 13. Audit Log
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entreprise_id UUID REFERENCES entreprise(id) ON DELETE SET NULL,
    utilisateur_id UUID REFERENCES utilisateur(id) ON DELETE SET NULL,
    utilisateur_nom VARCHAR(255),
    role VARCHAR(50),
    module VARCHAR(100) NOT NULL,
    action VARCHAR(100) NOT NULL,
    details TEXT,
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 14. Contact Lead (Site Vitrine)
CREATE TABLE contact_lead (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    nom VARCHAR(255) NOT NULL,
    entreprise_nom VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    telephone VARCHAR(50),
    message TEXT,
    statut VARCHAR(50) DEFAULT 'NOUVEAU',
    cree_le TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. High-Level Security & Multi-Tenant Architecture
- **Tenant Context Isolation**: A custom `TenantInterceptor` extracts `entreprise_id` from the decoded JWT claims and binds it to a `ThreadLocal<UUID> TenantContext`.
- **Repository Security**: Every query filters by `entreprise_id = TenantContext.getTenantId()`.
- **HMAC-SHA256 Signed QR Code**:
  Format: `SIMPLE3:AGENT:{agentId}:{timestamp}:{hmacSignature}`
  Verified using secret key in `HmacQrService`.
- **Geofencing Calculation**: Haversine distance formula computed in `PointageService`. If distance > `site.rayonGeofenceMetres`, standard pointage is saved with `anomalie = "ANOMALIE_GPS (Distance: X meters)"`.

---

## 4. UI/UX & Design Tokens
- **Palette**: Slate Dark (#0F172A), Emerald Accent (#10B981), Cyan Highlights (#06B6D4), Crimson Warnings (#EF4444).
- **Glassmorphism**: Semi-transparent card overlays, subtle backdrop blur (`backdrop-filter: blur(16px)`), luminous borders (`1px solid rgba(255, 255, 255, 0.1)`).
