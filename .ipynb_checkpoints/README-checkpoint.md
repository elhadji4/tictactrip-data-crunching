
# Tictactrip — Data Crunching (Tickets, Cities, Stations, Providers)

> Jupyter Notebook + Pandas — Exploration, Cleaning, Feature Engineering, KPIs & Visualisations

## 🎯 Objectifs

- **Prix**: calcul des prix **min / moyen / max** (global, par mode)
- **Durées**: min / moyenne / max **par trajet** (origine → destination)
- **Comparaisons par mode** (train / bus / carpooling) **selon la distance** (0–200, 201–800, 801–2000, 2000+ km)
- **Bonus**: visualisations, rapport des soucis de données, features additionnelles (heures/jours), distance Haversine, exports CSV

## 📦 Données utilisées

Les fichiers sont placés dans `DATA/` :

- `ticket_data.csv` — historique d'offres de billets (une ligne = une proposition)
- `cities.csv` — villes desservies (clé: `tickets.o_city` → `cities.id` ; noms dans `cities.local_name`)
- `stations.csv` — stations desservies (clé: `tickets.o_station`/`d_station` → `stations.id` ; noms dans `stations.unique_name`)
- `providers.csv` — informations providers (clé: `tickets.company` → `providers.id`; type de transport dans `providers.transport_type`)

> **Remarque colonnes** (observées dans les fichiers fournis):
> - `tickets`: `id, company, o_station, d_station, departure_ts, arrival_ts, price_in_cents, search_ts, o_city, d_city, middle_stations, other_companies`
> - `cities`: `id, local_name, unique_name, latitude, longitude, population`
> - `stations`: `id, unique_name, latitude, longitude`
> - `providers`: `id, company_id, provider_id, name, fullname, has_wifi, has_plug, has_adjustable_seats, has_bicycle, transport_type`

## 🗂️ Arborescence du projet

```
Projet_Tictactrip/
├── DATA/
│   ├── ticket_data.csv
│   ├── cities.csv
│   ├── stations.csv
│   ├── providers.csv
│   └── figures/                 # (générées par le notebook)
├── scripts.ipynb                # notebook principal (étapes 1 → 10)
└── README.md                    # ce fichier
```

## 🔧 Environnement & dépendances

- Python 3.x
- Packages: `pandas`, `numpy`, `matplotlib`
- (Optionnel) `IPython.display` pour l'affichage dans le notebook

Installation rapide (si besoin) :

```bash
pip install pandas numpy matplotlib
```

## 🚀 Comment exécuter

1. Placez les 4 fichiers CSV dans `DATA/`.
2. Ouvrez `scripts.ipynb` dans Jupyter.
3. Exécutez les cellules **dans l'ordre** (Étapes 1 → 10). 
4. Les figures seront générées dans `DATA/figures/`, et des CSV exportés dans `DATA/`.

## 🧱 Pipeline — Étapes détaillées

1. **Chargement** des datasets depuis `DATA/` (chemins relatifs)
2. **Nettoyage de base**: 
   - Conversion des timestamps (`departure_ts`, `arrival_ts`, `search_ts`) → `datetime`
   - Calcul du `price_eur = price_in_cents / 100`
3. **Merge providers** (clé: `tickets.company` ↔ `providers.id`) 
   - Renommage pour éviter conflits (`id` → `provider_key`, `provider_id` → `provider_external_id`)
   - Ajout: `provider_name`, `provider_fullname`, `transport_mode`, `has_wifi`, `has_plug`
4. **Merge cities** (origine + destination) 
   - `local_name` → `origin_city_name` / `dest_city_name`
   - Ajout des lat/lon (`origin_lat/lon`, `dest_lat/lon`) et populations
5. **Merge stations** (origine + destination, **LEFT JOIN** car beaucoup de NaN) 
   - Ajout des noms de stations et lat/lon si disponibles
6. **Features**: 
   - `duration_hours` (arrivée − départ en heures)
   - `advance_hours` (départ − recherche en heures)
   - `distance_km` (Haversine entre villes)
   - `distance_bucket` (0–200, 201–800, 801–2000, 2000+)
   - (Optionnel) `departure_hour`, `departure_dow`
7. **Filtrage minimal**: 
   - `duration_hours >= 0` et `price_eur > 0`
   - (Optionnel) **Nettoyage renforcé** (outliers): bornes sur durée, advance, distance et cap quantile pour le prix
8. **KPIs demandés**: 
   - **Prix min / moyen / max** (global, par `transport_mode`)
   - **Durée min / moyenne / max** par **trajet OD** (`origin_city_name` → `dest_city_name`)
   - **Prix & durée moyens** par **tranche de distance** et **mode**
9. **Visualisations**: 
   - Distribution des prix
   - Scatter **prix vs. advance** (+ moyenne par bins de 24h)
   - Boxplot **durée par mode**
   - Bar chart **prix moyen par tranche de distance & mode**
   - Heatmap **Top 20 flux OD**
10. **Exports**:
    - `DATA/tickets_clean.csv` (dataset final)
    - `DATA/kpi_duration_by_od.csv`
    - `DATA/kpi_by_distance_bucket_mode.csv`

## 📊 Exemples de sorties & indicateurs

> Les exemples ci-dessous sont issus des exécutions sur les données fournies.

- **Bornes observées (avant nettoyage renforcé)**:
  - `min_price ≈ 3 €`, `max_price ≈ 385.5 €`
  - `min_duration_h ≈ 0.33 h`, `max_duration_h ≈ 492.85 h` (outlier)
- **Taux de jointure provider**: proportion de lignes avec `provider_name` non nul
- **Paires OD complètes**: proportion de lignes avec `origin_city_name` & `dest_city_name`

**KPIs typiques** (nettoyés):
- Global: `price_eur` min/mean/max
- Par mode (`transport_mode`): `count`, `min`, `mean`, `max` des prix
- Par trajet OD: `n`, `distance_km` médiane, `duration_min/mean/max`, `price_mean`
- Par **tranche de distance** × **mode**: `n`, `price_mean`, `duration_mean`

**Figures générées** (dans `DATA/figures/`):
- `distribution_prix.png`, `prix_vs_advance.png`, `duree_par_mode.png`,
- `prix_moyen_par_bucket_mode.png`, `heatmap_top20_OD.png`

## ⚠️ Points d’attention & qualité des données

- **NaN fréquents** sur `o_station` / `d_station` → garder des `LEFT JOIN`, ne pas filtrer drastiquement
- `cities.local_name` (et non `name`) — s’assurer de renommer correctement lors des merges
- **Outliers** potentiels: durées très longues et distances extrêmes — appliquer des bornes métier si nécessaire
- **Types de clés**: harmoniser (`int`) pour les merges, enlever les doublons côté providers

## ➕ Pistes & Bonus

- **KPIs calendaires** (jour/semaine, heure) par mode
- **Modélisation**: régression simple `price ~ distance_km + advance_hours + C(transport_mode)`
- **Visualisations interactives** (Plotly) — selon contraintes d’environnement
- **Sourcing externe**: météo, jours fériés, événements, pour enrichir l’analyse

## 📤 Publication & rendu

- Pousser `scripts.ipynb`, `README.md` et les exports CSV/figures dans un **repo GitHub/GitLab**
- Envoyer le lien du repo à **bot+jobs@tictactrip.eu**
  - **Objet de l’email**: `DATA@NOM PRENOM`

## 📜 Licence

Ce projet est un exercice technique. Les données restent sous la licence et les conditions du fournisseur des datasets d'origine.

## 👋 Contact

Pour toute question sur ce notebook ou le code, ouvrez une issue sur le repo.
