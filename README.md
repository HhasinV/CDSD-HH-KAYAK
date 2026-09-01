# Projet Kayak — Pipeline de recommandation de destinations

> Construction d'un pipeline de données complet qui recommande les meilleures destinations de vacances en France, en croisant **données météo** et **données hôtelières**.
> Projet réalisé dans le cadre du bloc **Data Collection & Management** (Certification Jedha).

---

## Contexte

L'équipe marketing de Kayak souhaite recommander des destinations en France, mais ne dispose d'**aucune donnée**. La mission : collecter, nettoyer, stocker et restituer des données réelles sur la **météo** et les **hôtels** de 35 villes touristiques françaises, puis produire deux visualisations de recommandation.

---

## Architecture

Les données circulent de gauche à droite — chaque étape alimente la suivante :

```
   Nominatim              OpenWeather            Booking.com
 (coordonnées GPS) ──┬──▶  (météo 8 jours)      (scraping hôtels)
                     │            │                    │
                     └────────────┴──────────┬─────────┘
                                             ▼
                                    ┌──────────────────┐
                                    │   Amazon S3      │  Data Lake
                                    │  raw/ + curated/ │
                                    └────────┬─────────┘
                                             │  ETL (extract from S3)
                                             ▼
                                    ┌──────────────────┐
                                    │ PostgreSQL (RDS) │  Data Warehouse
                                    │ cities + hotels  │
                                    └────────┬─────────┘
                                             │
                                   ┌─────────┴─────────┐
                                   ▼                   ▼
                             Carte Plotly 1      Carte Plotly 2
                            (Top 5 villes)      (Top 20 hôtels)
```

**Contrainte d'orchestration :** le géocodage (Nominatim) est un préalable obligatoire — l'API météo a besoin des coordonnées pour fonctionner.

---

## Stack technique

| Domaine | Outils |
|---|---|
| **Collecte (API)** | `requests` · Nominatim · OpenWeatherMap (One Call 3.0) |
| **Scraping** | `Playwright` (navigateur headless, contenu JavaScript) |
| **Traitement** | `pandas` · `numpy` |
| **Data Lake** | AWS S3 (`boto3`) |
| **Data Warehouse** | AWS RDS PostgreSQL (`SQLAlchemy` · `psycopg2`) |
| **Visualisation** | `Plotly` |
| **Secrets** | `python-dotenv` |

---

## Structure du dépôt

> Adapte les noms de fichiers à ton organisation réelle.

```
.
├── notebooks/
│   ├── 01_geocodage.ipynb          # Nominatim → coordonnées GPS
│   ├── 02_meteo_score.ipynb        # OpenWeather → score de beau temps
│   ├── 03_scraping_hotels.ipynb    # Playwright → hôtels + coordonnées
│   └── 04_fusion_s3_etl.ipynb      # Fusion + S3 + ETL PostgreSQL + cartes
├── data/
│   ├── raw/                        # Données brutes (CSV + JSON)
│   └── curated/                    # kayak_enriched.csv (fichier enrichi)
├── .env                            # Secrets (NON versionné)
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Installation

### 1. Cloner le dépôt

```bash
git clone https://github.com/<utilisateur>/<depot>.git
cd <depot>
```

### 2. Environnement virtuel + dépendances

```bash
python -m venv .venv
source .venv/bin/activate          # Windows : .venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Installer le navigateur Playwright (pour le scraping)

```bash
playwright install chromium
playwright install-deps chromium   # Linux : dépendances système (sudo si besoin)
```


## Utilisation

Exécuter les notebooks **dans l'ordre** (chaque étape produit les fichiers utilisés par la suivante) :

| Ordre | Notebook | Produit |
|---|---|---|
| 1 | `01_geocodage.ipynb` | `data/raw/cities.csv` |
| 2 | `02_meteo_score.ipynb` | `data/raw/weather_scored.csv` + Carte 1 |
| 3 | `03_scraping_hotels.ipynb` | `data/raw/hotels.csv` |
| 4 | `04_fusion_s3_etl.ipynb` | `kayak_enriched.csv` → S3 → PostgreSQL + Carte 2 |

> Le scraping des 35 villes prend ~35-45 min ; un système de **cache par ville** rend l'opération reprenable en cas d'interruption.

---

## Détail du pipeline

1. **Géocodage** — Nominatim convertit 35 noms de villes en coordonnées GPS. Création d'un `city_id` unique qui relie toutes les données du projet.
2. **Météo & score** — OpenWeather fournit 8 jours de prévisions. Un **score de beau temps** combine 3 critères normalisés (température, ciel dégagé, probabilité de pluie).
3. **Scraping** — Playwright pilote un navigateur pour extraire les hôtels de Booking (contenu chargé en JavaScript). Coordonnées récupérées sur chaque fiche hôtel.
4. **Data Lake S3** — Fusion météo + hôtels via `city_id`, puis stockage brut (`raw/`) et nettoyé (`curated/`).
5. **ETL PostgreSQL** — Extraction **depuis S3**, chargement dans deux tables normalisées (`cities` + `hotels`) reliées par clé étrangère.
6. **Restitution** — Deux cartes Plotly : top 5 destinations, top 20 hôtels (via requête SQL avec `JOIN`).



## Auteur

**Henintsoa HASINAVALONA** — Certification Data Full Stack, Jedha Bootcamp.
