# Fog — Spécification de la v1

- **Date :** 2026-10-16
- **Statut :** en revue
- **Portée :** v1 (prototype solo, qualité pro)

---

## 1. Contexte et objectif

Fog est un « brouillard de guerre » dans la vraie vie : une carte du monde recouverte de brouillard, qui se dissipe là où l'utilisateur passe physiquement. Un pourcentage d'exploration est affiché à plusieurs niveaux de zone (arrondissement → ville → région → pays → monde).

**Public :** l'auteur et ses amis. Qualité attendue : professionnelle (code propre, testé, fiable). Une publication gratuite sur l'App Store et Google Play est envisagée plus tard.

**Critères de succès de la v1 :**

1. J'installe l'app, j'accepte la localisation, puis je l'oublie. Une semaine plus tard, le brouillard s'est dissipé le long de tous mes trajets.
2. Je vois mon pourcentage pour chaque niveau de zone, de l'arrondissement au monde, et les chiffres sont cohérents.
3. La consommation de batterie due au suivi est inférieure à 5 % par jour en usage normal.
4. L'app enregistre partout dans le monde. Les pourcentages sont disponibles pour les pays importés (la France en premier) et se mettent à jour rétroactivement quand un pays est importé.

## 2. Périmètre

### Inclus dans la v1

- App mobile iOS et Android (Expo / React Native, TypeScript).
- Suivi de position passif et continu en arrière-plan, sans action de l'utilisateur.
- Découverte par cellules H3 de résolution 12 (~9 m de côté), stockées uniquement sur le téléphone.
- Carte avec brouillard et position courante.
- Pourcentages par niveau de zone, calculés sur le téléphone.
- Backend NestJS + Fastify servant des données géographiques publiques (rues et zones), sans comptes.
- Import OSM pays par pays, la France en premier.

### Hors périmètre de la v1 (évolutions prévues, voir §12)

Comptes et connexion Google / Apple, synchronisation, classement entre amis, guildes et carte coopérative, map matching, sous-scores « rues / sentiers », module natif de suivi, suivi des plantages, interface web.

## 3. Décisions clés

| Sujet | Décision | Raison |
|---|---|---|
| Plateforme | Expo (React Native) + TypeScript, iOS et Android | Le suivi en arrière-plan exclut le web ; stack maîtrisée par l'auteur |
| Unité de découverte | Cellules H3 résolution 12 (~9 m) | Taille proche de la précision GPS ; une résolution plus fine n'apporte rien |
| Pourcentage | Cellules de rue découvertes / cellules de rue de la zone | Ressenti « j'ai exploré chaque rue » sans map matching |
| Voies comptées | Tout ce qui existe physiquement (voir §6.2) | Le score doit refléter ce qu'une personne parcourt réellement ; 100 % n'a pas besoin d'être atteignable |
| Déplacements | Tous les modes comptent (à pied, vélo, voiture, train…) | Choix produit ; seuls l'avion et les sauts GPS sont filtrés |
| Suivi | Passif en continu, via une interface `LocationSource` ; implémentation v1 avec `expo-location` | Gratuit ; la technologie de suivi reste remplaçable (module natif maison si la batterie l'exige) |
| Niveaux de zone | Arrondissement (s'il existe) → ville → région → pays → monde | Satisfaction locale et vision globale |
| Couverture | Monde entier, import pays par pays (France d'abord) | Importer la planète d'un coup est trop lourd |
| Backend | NestJS (adaptateur Fastify) + PostgreSQL / PostGIS / h3-pg | Structure prête pour les comptes, le classement et les guildes |
| Confidentialité | Le téléphone ne transmet jamais sa position précise ; aucune trace GPS brute n'est conservée | Une donnée de localisation est très sensible |
| Carte | MapLibre + fond OpenFreeMap (OSM) | Gratuit ; ce qui est affiché correspond à ce qui est compté |

## 4. Architecture globale

```
┌───────────────────────── TÉLÉPHONE (Expo / React Native) ─────────────────────────┐
│                                                                                    │
│  LocationSource ──► pending_points ──► Pipeline geo-core ──► discovered_cells      │
│  (expo-location,     (tampon durable)   filtres, cellules,    (SQLite)             │
│   tâche en arrière-plan)                interpolation, anneau      │               │
│                                                                    ▼               │
│                                         Carte MapLibre + brouillard   Calcul des % │
│                                                                          ▲         │
│                                                    tiles_cache / zones_cache       │
└──────────────────────────────────────────────────────────────────────────┼─────────┘
                                                                           │ GET /tiles/{tileId}
                                                                           │ GET /meta
┌──────────────────────────────── SERVEUR ─────────────────────────────────┼─────────┐
│   API NestJS + Fastify ◄──── PostgreSQL + PostGIS + h3-pg                          │
│                                     ▲                                              │
│   osm-importer (Node + osmium) ─────┘                                              │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Principe :** le téléphone enregistre, découvre et calcule. Le serveur ne fournit que des données géographiques publiques (cellules de rue et zones). Il ne connaît pas les déplacements de l'utilisateur.

### 4.1 Organisation du dépôt (monorepo pnpm)

```
fog/
├── apps/
│   ├── mobile/          # App Expo (React Native + TypeScript), Expo Router
│   └── api/             # API NestJS (adaptateur Fastify)
├── packages/
│   └── geo-core/        # Code partagé : H3, filtres, interpolation, calcul des %,
│                        # types et contrat de l'API
├── tools/
│   └── osm-importer/    # Import OSM → PostgreSQL
├── docker/
│   └── postgres/        # Dockerfile PostGIS + h3-pg
├── docker-compose.yml
└── docs/
```

`geo-core` est le seul endroit où l'on transforme une géométrie en cellules. L'app et l'importeur l'utilisent tous les deux, ce qui garantit des calculs identiques des deux côtés.

**Outils transverses :** TypeScript en mode strict partout, ESLint + Prettier, Vitest (geo-core, importeur), Jest (app, API), GitHub Actions.

## 5. App mobile

### 5.1 Suivi de position

- Interface interne `LocationSource` : `start()`, `stop()`, `isRunning()`, et un flux de lots de points `{ lat, lng, accuracy, speed, timestamp }`. Le reste de l'app ne dépend que de cette interface.
- Implémentation v1 : `ExpoLocationSource`, basée sur `expo-location` + `expo-task-manager` (`startLocationUpdatesAsync`).
  - Distance minimale entre deux points : ~10 m.
  - iOS : mise en pause automatique activée et mode d'arrière-plan « location ».
  - Android : service permanent avec une notification visible (« Fog explore… »).
- Une seconde implémentation, `ReplayLocationSource`, rejoue des fichiers GPX. Elle sert aux tests et au développement.
- **Autorisations :** un écran d'accueil explique le concept. L'app demande d'abord « Pendant l'utilisation », puis « Toujours ».
- **Robustesse :** la tâche en arrière-plan écrit d'abord les points reçus dans `pending_points`, ce qui est rapide et durable. Le traitement se fait ensuite. Il est idempotent et rejouable après un plantage. Les points sont supprimés dès qu'ils ont été traités.

### 5.2 Pipeline de découverte (`geo-core`, fonctions pures)

Le pipeline traite les points dans l'ordre chronologique :

1. Rejeter un point dont la précision annoncée dépasse 50 m.
2. Rejeter un point dont la vitesse, calculée depuis le dernier point accepté, dépasse 400 km/h (avion, saut GPS).
3. Convertir le point en cellule H3 de résolution 12.
4. **Interpoler :** entre deux points acceptés consécutifs, ajouter le chemin de cellules (`gridPathCells`), sauf si l'écart dépasse 2 km ou 5 minutes. Si `gridPathCells` échoue (cas rares près des pentagones H3), on n'interpole pas.
5. **Dévoiler un rayon :** ajouter le premier anneau de voisines (`gridDisk(cell, 1)`), soit un rayon d'environ 15 m.
6. Enregistrer les cellules sans écraser une date de première découverte existante (`INSERT OR IGNORE`).

Le dernier point accepté est conservé dans `tracker_state` pour assurer l'interpolation d'un lot à l'autre.

### 5.3 Stockage local (expo-sqlite, migrations versionnées)

| Table | Colonnes | Rôle |
|---|---|---|
| `discovered_cells` | `cell` TEXT PK, `tile` TEXT (indexé), `first_seen_at` INTEGER | Toutes les cellules découvertes |
| `fog_lod` | `res` INTEGER, `cell` TEXT, `r3` TEXT (indexé), PK(`res`, `cell`) | Cellules parentes découvertes aux résolutions 10, 8, 6 et 4, pour l'affichage dézoomé |
| `discovered_street_cells` | `cell` TEXT PK, `zone_id` TEXT (indexé) | Cellules découvertes qui sont des cellules de rue, avec leur zone la plus fine |
| `tiles_cache` | `tile` TEXT PK, `dataset_version`, `covered`, `payload` (JSON), `fetched_at` | Tuiles reçues de l'API |
| `zones_cache` | `zone_id` PK, `name`, `level`, `parent_id`, `country_code`, `street_cell_count`, `dataset_version` | Zones connues |
| `pending_tiles` | `tile` PK, `attempts`, `next_attempt_at` | Tuiles à télécharger |
| `pending_points` | `id` PK, `lat`, `lng`, `accuracy`, `speed`, `timestamp` | Tampon des points bruts |
| `tracker_state` | clé / valeur | Dernier point accepté, métadonnées de `/meta` |

Les identifiants H3 sont stockés en texte hexadécimal, car les entiers JavaScript perdent en précision au-delà de 53 bits. La colonne `tile` est la cellule parente de résolution 7. `fog_lod` est alimentée à chaque insertion dans `discovered_cells`.

### 5.4 Tuiles et calcul des pourcentages

- À chaque nouvelle cellule découverte, l'app vérifie que sa tuile (parente de résolution 7) est en cache. Si ce n'est pas le cas, la tuile est ajoutée à `pending_tiles`.
- Quand une tuile arrive, l'app intersecte ses cellules de rue avec les `discovered_cells` de cette tuile et remplit `discovered_street_cells`. Les découvertes suivantes dans une tuile déjà en cache sont vérifiées directement dans son contenu.
- **Pourcentage d'une zone Z :** nombre de `discovered_street_cells` dont la zone est Z ou une zone descendante de Z, divisé par le `street_cell_count` de Z.
- **Monde :** nombre total de `discovered_street_cells` divisé par `worldStreetCellEstimate` (reçu via `/meta`), affiché avec la mention « estimation ».
- Une cellule découverte qui n'est pas une cellule de rue (un champ, un toit) dissipe le brouillard mais ne compte pas dans les pourcentages.
- Si `/meta` annonce une nouvelle `datasetVersion`, les tuiles en cache sont marquées comme périmées et retéléchargées progressivement. `discovered_street_cells` est alors recalculée tuile par tuile.

### 5.5 Écrans (Expo Router)

1. **Accueil :** concept, explication de la localisation, demande des autorisations.
2. **Carte** (écran principal) : brouillard, position courante, bandeau si une autorisation manque.
3. **Progression :** pourcentages de la zone courante, de l'arrondissement au monde. Mention « bientôt disponible » pour un pays non importé, « estimation » pour le monde, et date de dernière mise à jour.
4. **Réglages :** mettre le suivi en pause ou le reprendre, volume de données locales, tout effacer (avec confirmation), mentions OSM et OpenFreeMap.

## 6. Backend et données

### 6.1 Import OSM (`tools/osm-importer`)

Commande : `pnpm import --country fr`. L'importeur peut être relancé sans risque.

1. Télécharger l'extrait Geofabrik du pays (`france-latest.osm.pbf`, ~4 Go).
2. Filtrer avec `osmium tags-filter` : voies (`highway=*`) et limites administratives (`boundary=administrative`).
3. Exporter avec `osmium export` en GeoJSON ligne par ligne. Osmium reconstruit les polygones des limites.
4. Charger les limites dans `zones` (géométrie PostGIS) selon la table de niveaux du pays (§6.3). Calculer les parents par inclusion spatiale.
5. Lire les voies en flux et calculer leurs cellules avec `geo-core` :
   - **ligne :** densifier tous les ~4 m, convertir chaque point en cellule, dédoublonner ;
   - **surface** (place, zone piétonne) : cellules dont le centre est dans le polygone (`polygonToCells`), plus les cellules du contour.
6. Charger les cellules en masse (`COPY`) dans une table de travail.
7. Rattacher chaque cellule à sa zone la plus fine (centre de la cellule dans le polygone). Les cellules hors du pays, présentes dans la marge de l'extrait Geofabrik, sont écartées.
8. Calculer `street_cell_count` par zone la plus fine, puis cumuler vers les zones parentes.
9. Dans une transaction, remplacer les données du pays et incrémenter `datasetVersion`.

**Prérequis :** `osmium-tool` (fourni via Docker) et ~50 Go d'espace disque libre pour la France.

### 6.2 Voies comptées

Le score doit refléter tout ce qu'une personne peut physiquement parcourir.

- **Comptées :** toutes les voies `highway=*`, y compris autoroutes, rues, ruelles, rues piétonnes, pistes cyclables, trottoirs, allées, passages, escaliers, sentiers (`path`), chemins (`track`) et voies privées. Les surfaces (`area=yes`, `area:highway=*`), comme les places et parvis, sont remplies de cellules.
- **Exclues :** uniquement ce qui n'existe pas physiquement : `highway=proposed`, `highway=construction`, `highway=razed`, `highway=abandoned`, ainsi que `abandoned:highway=*` et `razed:highway=*`. Une voie en construction sera comptée à l'import suivant, une fois ouverte.
- La liste est définie dans un fichier de configuration de l'importeur.

### 6.3 Zones

Une table de niveaux par pays associe les `admin_level` OSM aux niveaux de Fog. Un niveau absent est simplement ignoré.

| Pays | country | region | city | district |
|---|---|---|---|---|
| `fr` | 2 | 4 | 8 | 9 (Paris, Lyon, Marseille) |
| `mc` (tests) | 2 | — | — | — |

**Monde :** `worldStreetCellEstimate` = 64 millions de km de routes dans le monde × ratio mesuré sur les pays importés. Ce ratio est le nombre de cellules de rue comptées (sentiers compris), divisé par le nombre de km de **routes** du pays (hors `path`, `track`, `footway`, `cycleway`, `steps`, `bridleway`). On compare ainsi des grandeurs de même nature avec le chiffre mondial, qui ne compte que les routes. La valeur est recalculée à chaque import. Elle sera remplacée par un total exact quand la couverture le permettra.

### 6.4 Schéma PostgreSQL (PostGIS + h3-pg)

- `zones` (`id` TEXT PK, `osm_id` BIGINT, `level` ENUM(`country`, `region`, `city`, `district`), `name`, `parent_id`, `country_code`, `street_cell_count` BIGINT, `geom` geometry(MultiPolygon, 4326))
- `street_cells` (`cell` h3index PK, `tile` h3index indexé, `zone_id`, `country_code`)
- `datasets` (`version` INTEGER, `country_code`, `osm_timestamp`, `imported_at`)
- `meta` (`dataset_version`, `world_street_cell_estimate`)

L'image Docker est construite sur `postgis/postgis`, avec l'extension h3-pg ajoutée. Les migrations sont versionnées.

### 6.5 API (NestJS + Fastify, sans authentification)

Les types de réponse sont définis dans `geo-core` et partagés avec l'app.

**`GET /tiles/{tileId}`**

```ts
type TileResponse = {
  tileId: string;          // cellule H3 de résolution 7
  datasetVersion: number;
  covered: boolean;        // false = pays non importé (ce n'est pas une erreur)
  cellsByZone: Record<string, string[]>; // zone la plus fine → cellules de rue
  zones: Zone[];           // zones présentes dans la tuile, avec toutes leurs parentes
};
type Zone = {
  id: string; osmId: number; level: 'country' | 'region' | 'city' | 'district';
  name: string; parentId: string | null; countryCode: string; streetCellCount: number;
};
```

**`GET /meta`** → `{ datasetVersion, worldStreetCellEstimate, countries: [{ code, osmTimestamp, importedAt }] }`

- **Validation :** `tileId` doit être un index H3 valide de résolution 7. Sinon, l'API répond `400 INVALID_TILE_ID`.
- **Format d'erreur :** `{ error: { code, message } }`.
- **Cache :** en-têtes `Cache-Control: public, max-age=86400` et `ETag` (`datasetVersion` + `tileId`). Réponses compressées.
- **Limitation de débit :** `@nestjs/throttler`, avec un `429` explicite.
- **Taille attendue :** quelques milliers de cellules au plus pour une tuile urbaine dense, en JSON compressé. Un format binaire sera envisagé si nécessaire.

**Exécution :** en local sur l'ordinateur de l'auteur au début. L'app tolère que le serveur soit injoignable (§8). Pour des testeurs externes, il faudra un hébergement ou un tunnel (Cloudflare Tunnel).

## 7. Rendu du brouillard

- **Carte :** `@maplibre/maplibre-react-native`, avec le fond OpenFreeMap. Attributions OSM et OpenFreeMap affichées.
- **Calque :** un `FillLayer` dont le polygone couvre la zone visible, avec une marge, et qui est **percé de trous** aux cellules découvertes. Les trous sont obtenus avec `cellsToMultiPolygon` de `h3-js`.
- **Bord adouci :** un `LineLayer` avec `line-blur` sur le contour des trous.
- **Niveau de détail selon le zoom** (seuils à ajuster) :

| Zoom | Résolution affichée | Source |
|---|---|---|
| ≥ 15 | 12 | `discovered_cells`, filtrée par les tuiles visibles |
| 13–14 | 10 | `fog_lod` |
| 10–12 | 8 | `fog_lod` |
| 6–9 | 6 | `fog_lod`, filtrée par les parentes `r3` visibles |
| < 6 | 4 | `fog_lod` (volume très faible, chargé en entier) |

- **Mise à jour :** au repos de la caméra (`onRegionDidChange`) et quand de nouvelles cellules arrivent. Le calcul est fait hors du rendu, avec un cache par (tuile ou parente, résolution).
- **Objectifs :** 60 images/s pendant le déplacement de la carte, et une mise à jour du brouillard en moins de 200 ms.

## 8. Gestion des erreurs

| Situation | Comportement |
|---|---|
| Localisation refusée ou limitée à « Pendant l'utilisation » | Bandeau explicatif et bouton vers les réglages. La carte reste utilisable |
| Pas de GPS ou pas de signal | Rien à faire. Le suivi reprend seul |
| Point aberrant | Filtré (§5.2) |
| API injoignable ou erreur 5xx | Derniers pourcentages connus et « mis à jour il y a X ». La tuile reste dans `pending_tiles`, avec un délai croissant entre les tentatives (plafonné à 1 h) |
| `covered: false` | « Bientôt disponible ». Nouvelle demande quand `/meta` change |
| 429 | Les demandes de tuiles sont espacées davantage |
| Nouvelle `datasetVersion` | Rafraîchissement progressif des tuiles (§5.4) |
| Plantage pendant un traitement | `pending_points` et l'idempotence garantissent qu'aucun point n'est perdu ni compté deux fois |
| Évolution du schéma local | Migrations SQLite versionnées |

Pas d'outil de suivi des plantages en v1 : un rapport de plantage pourrait contenir une position.

## 9. Confidentialité, sécurité et licences

- **Sur le téléphone :** cellules découvertes et dates. Les points bruts sont supprimés dès qu'ils ont été traités.
- **Ce qui sort du téléphone :**
  - les identifiants de tuiles de résolution 7 (~5 km²) et l'adresse IP, vers l'API ;
  - les requêtes de fond de carte vers OpenFreeMap, qui révèlent la zone affichée ;
  - rien d'autre.
- **Sauvegarde :** assurée par les sauvegardes du système (iCloud, sauvegarde Android). Quelques Mo, même après des années d'usage.
- **Effacement :** « Tout effacer » vide toutes les tables locales.
- **API :** lecture seule, validation stricte des entrées, limitation de débit, aucune donnée utilisateur.
- **Licences :** les données OSM sont sous ODbL. La mention « © OpenStreetMap contributors » est obligatoire dans l'app. Les cellules de rue forment une base dérivée : en cas de publication publique, elle devra être proposée sous ODbL. Point à vérifier avant toute publication.

## 10. Stratégie de tests

| Composant | Outils | Contenu |
|---|---|---|
| `geo-core` | Vitest | Filtres, conversion en cellules, interpolation, anneau, calcul des %, densification des lignes, remplissage des surfaces. Tests de référence sur de vraies traces GPX (marche, voiture, tunnel, vol) |
| App | Jest + React Native Testing Library | Composants et hooks. Chaîne complète de découverte avec `ReplayLocationSource` et une base SQLite de test. Migrations |
| API | Jest + `fastify.inject` + Testcontainers (PostGIS + h3-pg) | Contrat de `/tiles` et `/meta`, validation, cache, erreurs |
| Importeur | Vitest + extrait OSM de Monaco | Import complet, totaux attendus stables, idempotence |
| Intégration continue | GitHub Actions | Lint, vérification des types, tous les tests, à chaque push |
| Conditions réelles | Checklist manuelle | iPhone et Android : semaine de mesure de batterie, app tuée, redémarrage, mode avion, autorisations refusées ou rétrogradées |

## 11. Risques et points ouverts

1. **h3-js sur Hermes (prioritaire) :** vérifier le fonctionnement et les performances de `h3-js` dans le moteur JavaScript de React Native. C'est la toute première tâche du plan. En cas d'échec, utiliser un binding natif de H3.
2. **Fiabilité du suivi passif avec `expo-location` :** mise en pause par iOS, fermeture forcée, économiseurs de batterie Android. À mesurer en conditions réelles. Si l'objectif de batterie (< 5 %/jour) ou la fiabilité ne sont pas atteints, développer le module natif maison (§12).
3. **Volume de l'import France :** plus de 100 millions de cellules attendues, peut-être le double avec les sentiers et chemins. À mesurer. Partitionner `street_cells` par pays si nécessaire.
4. **Performance du brouillard** avec plusieurs millions de cellules découvertes : couverte par `fog_lod`, à valider sur un appareil d'entrée de gamme.
5. **ODbL** en cas de publication (§9).
6. **Validation par les stores** de la localisation en arrière-plan : formulaire de justification et vidéo demandés par Google Play. Concerne la publication, pas la v1.

## 12. Évolutions prévues

La v1 ne les construit pas, mais ne les bloque pas.

- **Comptes et connexion Google + Apple** (Sign in with Apple est obligatoire sur l'App Store dès qu'une connexion tierce est proposée), avec synchronisation des cellules vers le backend. Les cellules ne font que s'ajouter, donc la synchronisation est une simple union d'ensembles, sans conflit.
- **Classement entre amis**, basé uniquement sur les scores.
- **Guildes et carte coopérative.**
- **Zones privées**, pour masquer le domicile quand des cartes seront partagées.
- **Map matching**, pour un pourcentage au mètre de rue près, calculé en différé.
- **Sous-scores** (rues / sentiers) et autres indicateurs (pays et villes explorés).
- **Module natif de suivi**, en Swift et Kotlin derrière `LocationSource` : CoreMotion, détection de déplacement important, reconnaissance d'activité Android.
- **Import d'autres pays, puis du monde entier.**
- **Tests de bout en bout de l'interface avec Maestro**, et suivi des plantages respectueux de la vie privée.
