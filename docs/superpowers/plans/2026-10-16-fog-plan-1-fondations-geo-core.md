# Fog — Plan 1 : fondations et geo-core — plan d'implémentation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mettre en place le monorepo, valider `h3-js` sur Hermes et livrer `@fog/geo-core`, la bibliothèque partagée et testée qui transforme des points GPS en cellules découvertes, des géométries OSM en cellules de rue, et des comptes en pourcentages.

**Architecture:** monorepo pnpm. `packages/geo-core` est un paquet TypeScript interne sans étape de build : `main` pointe vers `src/index.ts`, et Metro comme Vitest consomment directement le TypeScript. Toute la logique est faite de fonctions pures autour de `h3-js`. `apps/mobile` est créé ici uniquement pour le test de faisabilité Hermes.

**Tech Stack:** Node 22, pnpm 9, TypeScript (strict), h3-js v4, Vitest, ESLint (flat config) + Prettier, Expo (SDK le plus récent via `create-expo-app@latest`), GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-16-fog-v1-design.md`

**Feuille de route des plans** (chacun livre un logiciel testable ; chaque plan est rédigé à la fin du précédent) :
1. **Fondations + geo-core** (ce plan)
2. Données + API : Docker PostGIS/h3-pg, `osm-importer` (Monaco puis France), API NestJS `/tiles` et `/meta`
3. App mobile : `LocationSource`, SQLite, pipeline de découverte, synchronisation des tuiles, écran Progression
4. Carte + brouillard : MapLibre, rendu à plusieurs niveaux de détail, accueil, checklist de tests en conditions réelles

## Global Constraints

- Node `22` (`.nvmrc`), pnpm `9`, TypeScript `strict: true` partout.
- Identifiants H3 : chaînes hexadécimales **en minuscules**, sur 15 caractères. Jamais de `bigint` ni de `number`.
- Coordonnées : `GeoPoint` utilise `{ lat, lng }`. Les géométries GeoJSON utilisent `[lng, lat]`. Horodatages en millisecondes depuis l'epoch.
- Constantes de la spec (§5.2, §5.3, §6.1), à copier telles quelles dans `packages/geo-core/src/constants.ts` :
  `DISCOVERY_RES = 12`, `TILE_RES = 7`, `LOD_RESOLUTIONS = [10, 8, 6, 4]`, `LOD_FILTER_RES = 3`, `MAX_ACCURACY_M = 50`, `MAX_SPEED_KMH = 400`, `MAX_INTERP_DISTANCE_M = 2000`, `MAX_INTERP_GAP_MS = 300_000`, `REVEAL_RING = 1`, `DENSIFY_STEP_M = 4`, `EARTH_RADIUS_M = 6_371_008.8`.
- Code, identifiants et commentaires en anglais. Textes visibles par l'utilisateur en français.
- Messages de commit au format Conventional Commits, terminés par la ligne `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Travail sur une branche dédiée, jamais directement sur `main`.

## Review Focus

1. **Lot de points désordonné ou en double** (l'OS peut relivrer des points) → les points sont triés, les doublons d'horodatage écartés, aucune cellule fantôme. *Test : Task 3.*
2. **Frontière entre deux lots** → traiter une trace en deux lots, en passant l'état du premier, donne exactement les mêmes cellules qu'en un seul lot (l'interpolation traverse la frontière). *Test : Task 4.*
3. **Pentagone H3 ou échec de `gridPathCells`** → pas de plantage, simplement pas d'interpolation sur ce segment. *Test : Task 4.*
4. **Zone de total 0 ou estimation mondiale absente** → ratio `0` ou `null`, jamais `NaN` ni `Infinity`. *Test : Task 6.*
5. **Géométries OSM dégénérées** (ligne d'un seul point, coordonnées consécutives en double, anneau de moins de 4 positions) → pas de plantage, résultat sensé. *Test : Task 7.*

---

### Task 1 : socle du monorepo et squelette de geo-core

**Files:**
- Create: `package.json`, `pnpm-workspace.yaml`, `.npmrc`, `.nvmrc`, `.gitignore`, `tsconfig.base.json`, `eslint.config.mjs`, `.prettierrc.json`, `.github/workflows/ci.yml`
- Create: `packages/geo-core/package.json`, `packages/geo-core/tsconfig.json`, `packages/geo-core/vitest.config.ts`, `packages/geo-core/src/index.ts`, `packages/geo-core/src/constants.ts`
- Test: `packages/geo-core/test/constants.test.ts`

**Interfaces:**
- Produces: le paquet `@fog/geo-core` (`"main": "src/index.ts"`, `"types": "src/index.ts"`), qui réexporte tous les modules. Les scripts racine `pnpm lint`, `pnpm typecheck` et `pnpm test` s'exécutent en `pnpm -r --if-present run <script>`. Chaque paquet définit `lint`, `typecheck` (`tsc --noEmit`) et `test`.

- [ ] **Step 1 : créer la configuration racine**

`pnpm-workspace.yaml` : `packages: ["apps/*", "packages/*", "tools/*"]`. `.npmrc` : `node-linker=hoisted` (exigé par Expo/Metro avec pnpm). `.nvmrc` : `22`. `tsconfig.base.json` : `strict`, `noUncheckedIndexedAccess`, `module`/`moduleResolution` `"Bundler"`, `target` `"ES2022"`. ESLint flat config avec `typescript-eslint` (preset `recommended`) et `eslint-config-prettier`, en ignorant `apps/mobile` (qui a sa propre config Expo).

- [ ] **Step 2 : écrire le test qui échoue** dans `packages/geo-core/test/constants.test.ts`

```ts
import { describe, expect, it } from 'vitest';
import * as C from '../src';

describe('constants', () => {
  it('match the spec values', () => {
    expect(C.DISCOVERY_RES).toBe(12);
    expect(C.TILE_RES).toBe(7);
    expect(C.LOD_RESOLUTIONS).toEqual([10, 8, 6, 4]);
    expect(C.LOD_FILTER_RES).toBe(3);
    expect(C.MAX_ACCURACY_M).toBe(50);
    expect(C.MAX_SPEED_KMH).toBe(400);
    expect(C.MAX_INTERP_DISTANCE_M).toBe(2000);
    expect(C.MAX_INTERP_GAP_MS).toBe(300_000);
    expect(C.REVEAL_RING).toBe(1);
    expect(C.DENSIFY_STEP_M).toBe(4);
  });
});
```

- [ ] **Step 3 : lancer le test pour vérifier qu'il échoue**

Run: `pnpm install && pnpm --filter @fog/geo-core test`
Expected: FAIL (exports manquants)

- [ ] **Step 4 : implémenter `constants.ts`** avec les valeurs des Global Constraints, en `as const`, réexporté depuis `src/index.ts`. Ajouter `h3-js` (v4) aux dépendances de geo-core.

- [ ] **Step 5 : vérifier toute la chaîne**

Run: `pnpm test && pnpm lint && pnpm typecheck`
Expected: les trois commandes passent.

- [ ] **Step 6 : écrire `.github/workflows/ci.yml`**

Déclenché sur `push` et `pull_request`. Étapes : `pnpm/action-setup` (version 9), `actions/setup-node` (node-version-file `.nvmrc`, cache pnpm), `pnpm install --frozen-lockfile`, `pnpm lint`, `pnpm typecheck`, `pnpm test`.

- [ ] **Step 7 : commit**

```bash
git add -A && git commit -m "chore: scaffold pnpm monorepo and geo-core package"
```

---

### Task 2 : test de faisabilité h3-js sur Hermes (prioritaire, spec §11.1)

**Files:**
- Create: `apps/mobile/` (via `create-expo-app`), `apps/mobile/app/h3-spike.tsx` (jetable)
- Create: `docs/superpowers/spikes/2026-10-16-h3-hermes.md` (résultats)
- Create: `scripts/h3-spike-expected.mjs` (jetable)

**Interfaces:**
- Produces: `apps/mobile` (paquet `mobile`) avec un `app/_layout.tsx` et un `app/index.tsx` minimaux (texte « Fog »), et une décision documentée : *h3-js utilisable sur Hermes : oui / non*.

- [ ] **Step 1 : créer l'app**

Run: `pnpm dlx create-expo-app@latest apps/mobile --template default --no-install`, puis `pnpm install` à la racine. Lancer le script `reset-project` du template, supprimer `app-example/` et nommer le paquet `mobile`. Ajouter `h3-js` aux dépendances de `apps/mobile`.

- [ ] **Step 2 : générer les valeurs attendues côté Node**

`scripts/h3-spike-expected.mjs` affiche en JSON, calculés avec h3-js dans Node :
- `latLngToCell(48.8634, 2.3601, 12)` ;
- `gridDisk(<cette cellule>, 1)` trié ;
- `gridPathCells` entre `latLngToCell(48.8634, 2.3601, 12)` et `latLngToCell(48.8643, 2.3601, 12)` ;
- le nombre d'anneaux de `cellsToMultiPolygon(gridDisk(<cellule>, 40))`.

Run: `node scripts/h3-spike-expected.mjs`
Expected: un objet JSON avec ces quatre valeurs.

- [ ] **Step 3 : écrire l'écran `app/h3-spike.tsx`**

Les valeurs attendues de l'étape 2 sont recopiées en dur. L'écran affiche :
- (a) `Hermes: oui/non` (`typeof (globalThis as { HermesInternal?: unknown }).HermesInternal === 'object'`) ;
- (b) PASS/FAIL pour chacune des 4 valeurs ;
- (c) des temps mesurés avec `performance.now()` :
  - 10 000 `latLngToCell` (seuil **< 500 ms**) ;
  - 1 000 `gridDisk(c, 1)` ;
  - 1 000 `gridPathCells` sur environ 100 m ;
  - `cellsToMultiPolygon` sur `gridDisk(c, 40)`, soit 4 921 cellules (seuil **< 200 ms**, le budget de rendu du brouillard).

Ajouter un lien vers cet écran depuis `app/index.tsx`.

- [ ] **Step 4 : lancer sur le simulateur iOS (Expo Go, Hermes)**

Run: `pnpm --filter mobile exec expo start --ios`, puis ouvrir l'écran du test.
Expected : `Hermes: oui`, 4 × PASS, les deux seuils respectés.
Si une erreur `TextDecoder is not defined` apparaît, ajouter un polyfill (`text-encoding`) importé en premier dans `app/_layout.tsx`, puis relancer.

- [ ] **Step 5 : consigner les résultats** dans `docs/superpowers/spikes/2026-10-16-h3-hermes.md` : appareil, version d'Expo, PASS/FAIL, temps mesurés et décision. **Si FAIL ou seuils dépassés : s'arrêter et faire remonter à l'utilisateur** avant les plans 3 et 4. Les Tasks 3 à 8 de ce plan restent valables, car elles tournent dans Node.

- [ ] **Step 6 : supprimer le code jetable** (`app/h3-spike.tsx`, le lien dans `app/index.tsx`, `scripts/h3-spike-expected.mjs`), puis vérifier.

Run: `pnpm --filter mobile exec tsc --noEmit && pnpm --filter mobile run lint`
Expected: les deux passent.

- [ ] **Step 7 : commit**

```bash
git add -A && git commit -m "chore(mobile): scaffold Expo app and record h3-js Hermes spike results"
```

---

### Task 3 : distance et filtres de points (spec §5.2, étapes 1-2)

**Files:**
- Create: `packages/geo-core/src/types.ts`, `packages/geo-core/src/distance.ts`, `packages/geo-core/src/filters.ts`
- Test: `packages/geo-core/test/distance.test.ts`, `packages/geo-core/test/filters.test.ts`

**Interfaces:**
- Produces:
  - `type GeoPoint = { lat: number; lng: number; accuracy: number; timestamp: number }`
  - `haversineMeters(a: { lat: number; lng: number }, b: { lat: number; lng: number }): number`
  - `filterPoints(points: GeoPoint[], lastAccepted: GeoPoint | null): GeoPoint[]` : renvoie les points acceptés, triés par `timestamp`.

- [ ] **Step 1 : écrire les tests qui échouent**

```ts
// distance.test.ts
it('measures 0.001° of latitude as ~111.2 m', () => {
  expect(haversineMeters({ lat: 48.86, lng: 2.36 }, { lat: 48.861, lng: 2.36 })).toBeCloseTo(111.2, 1);
});
it('returns 0 for identical points', () => {
  expect(haversineMeters({ lat: 1, lng: 1 }, { lat: 1, lng: 1 })).toBe(0);
});

// filters.test.ts
const p = (lat: number, t: number, accuracy = 10): GeoPoint => ({ lat, lng: 2.36, accuracy, timestamp: t });
const KM_NORTH = 1000 / 111_195; // degrés de latitude pour 1 km

it('drops points less accurate than 50 m and keeps exactly 50 m', () => {
  expect(filterPoints([p(48.86, 0, 60), p(48.86, 1000, 50)], null)).toEqual([p(48.86, 1000, 50)]);
});
it('drops a point implying more than 400 km/h and keeps comparing to the last accepted one', () => {
  const a = p(48.86, 0);
  const jump = p(48.86 + KM_NORTH, 5_000);   // 1 km en 5 s = 720 km/h
  const c = p(48.86 + 0.00045, 10_000);      // ~50 m en 10 s depuis a
  expect(filterPoints([a, jump, c], null)).toEqual([a, c]);
});
it('compares the first point of a batch to lastAccepted', () => {
  expect(filterPoints([p(48.86 + KM_NORTH, 5_000)], p(48.86, 0))).toEqual([]);
});
it('sorts out-of-order points and drops duplicate or past timestamps', () => {
  const a = p(48.86, 0), b = p(48.8601, 10_000);
  expect(filterPoints([b, a, { ...b }], null)).toEqual([a, b]);
  expect(filterPoints([a], p(48.86, 0))).toEqual([]);   // pas plus récent que lastAccepted
});
```

- [ ] **Step 2 : lancer les tests pour vérifier qu'ils échouent**

Run: `pnpm --filter @fog/geo-core test`
Expected: FAIL (modules introuvables)

- [ ] **Step 3 : implémenter.** `haversineMeters` utilise `EARTH_RADIUS_M`. `filterPoints` : trier une copie par `timestamp`, puis parcourir en gardant une référence `ref = lastAccepted`. Rejeter si `accuracy > MAX_ACCURACY_M`, si `timestamp <= ref.timestamp`, ou si la vitesse `haversine / Δt` dépasse `MAX_SPEED_KMH / 3.6` m/s. Sinon, accepter et faire `ref = point`.

- [ ] **Step 4 : lancer les tests pour vérifier qu'ils passent**

Run: `pnpm --filter @fog/geo-core test`
Expected: PASS

- [ ] **Step 5 : commit**

```bash
git add -A && git commit -m "feat(geo-core): add haversine distance and GPS point filters"
```

---

### Task 4 : découverte des cellules et traitement d'un lot (spec §5.2, étapes 3-6)

**Files:**
- Create: `packages/geo-core/src/discovery.ts`
- Test: `packages/geo-core/test/discovery.test.ts`

**Interfaces:**
- Consumes: `GeoPoint`, `filterPoints`, `haversineMeters` (Task 3).
- Produces:
  - `discoverCells(points: GeoPoint[], previous: GeoPoint | null): Map<string, number>` : de la cellule vers son horodatage de première découverte. Les points sont supposés déjà filtrés et triés.
  - `type BatchResult = { cells: Map<string, number>; lastAccepted: GeoPoint | null }`
  - `processBatch(points: GeoPoint[], lastAccepted: GeoPoint | null): BatchResult` = `filterPoints` puis `discoverCells`. `lastAccepted` vaut le dernier point accepté, ou l'entrée inchangée si aucun point n'est accepté.

**Règles d'horodatage :**
- la cellule d'un point et son anneau prennent le `timestamp` de ce point ;
- les cellules interpolées d'un segment (et leurs anneaux) prennent le `timestamp` du point d'arrivée du segment ;
- en cas de collision, on garde le minimum.

- [ ] **Step 1 : écrire les tests qui échouent**

```ts
const pt = (lat: number, lng: number, t: number): GeoPoint => ({ lat, lng, accuracy: 10, timestamp: t });
const A = pt(48.8634, 2.3601, 0);

it('reveals the point cell and its first ring (7 cells)', () => {
  const cells = discoverCells([A], null);
  expect([...cells.keys()].sort()).toEqual(gridDisk(latLngToCell(A.lat, A.lng, 12), 1).sort());
  expect(new Set(cells.values())).toEqual(new Set([0]));
});
it('interpolates between close consecutive points', () => {
  const B = pt(48.8643, 2.3601, 60_000); // ~100 m, 60 s
  const cells = discoverCells([A, B], null);
  for (const c of gridPathCells(latLngToCell(A.lat, A.lng, 12), latLngToCell(B.lat, B.lng, 12))) {
    expect(cells.has(c)).toBe(true);
  }
});
it('does not interpolate beyond 2 km', () => {
  expect(discoverCells([A, pt(48.8904, 2.3601, 600_000)], null).size).toBe(14); // ~3 km
});
it('does not interpolate beyond 5 minutes', () => {
  expect(discoverCells([A, pt(48.8679, 2.3601, 360_000)], null).size).toBe(14); // ~500 m, 6 min
});
it('keeps the earliest timestamp on collisions', () => {
  const cells = discoverCells([A, pt(A.lat, A.lng + 0.00001, 30_000)], null);
  expect(cells.get(latLngToCell(A.lat, A.lng, 12))).toBe(0);
});
it('gives the same cells whether a trace is processed in one or two batches', () => {
  const trace = Array.from({ length: 20 }, (_, i) => pt(48.86 + i * 0.0001, 2.3601, i * 8_000));
  const whole = processBatch(trace, null);
  const first = processBatch(trace.slice(0, 10), null);
  const second = processBatch(trace.slice(10), first.lastAccepted);
  expect(new Set([...first.cells.keys(), ...second.cells.keys()])).toEqual(new Set(whole.cells.keys()));
  expect(second.lastAccepted).toEqual(trace[19]);
});
it('does not throw around an H3 pentagon', () => {
  const [lat, lng] = cellToLatLng(getPentagons(12)[0]!);
  expect(() => discoverCells([pt(lat - 0.001, lng, 0), pt(lat + 0.001, lng, 60_000)], null)).not.toThrow();
});
it('returns lastAccepted unchanged when every point is rejected', () => {
  expect(processBatch([{ ...A, accuracy: 99 }], A).lastAccepted).toBe(A);
});
```

- [ ] **Step 2 : lancer les tests pour vérifier qu'ils échouent**

Run: `pnpm --filter @fog/geo-core test discovery`
Expected: FAIL

- [ ] **Step 3 : implémenter `discoverCells` et `processBatch`** dans `discovery.ts`. Parcourir les paires `(previous ?? null, p0), (p0, p1), …`. Interpoler avec `gridPathCells` seulement si `haversineMeters ≤ MAX_INTERP_DISTANCE_M` et `Δt ≤ MAX_INTERP_GAP_MS`. Entourer `gridPathCells` d'un `try/catch` : en cas d'exception, pas d'interpolation. Ajouter `gridDisk(c, REVEAL_RING)` pour chaque cellule (de point ou interpolée). La cellule de `previous` n'est pas réémise, sauf si elle fait partie d'un chemin.

- [ ] **Step 4 : lancer les tests pour vérifier qu'ils passent**

Run: `pnpm --filter @fog/geo-core test`
Expected: PASS

- [ ] **Step 5 : commit**

```bash
git add -A && git commit -m "feat(geo-core): discover cells with interpolation and reveal ring"
```

---

### Task 5 : hiérarchie des cellules et contrat de l'API (spec §5.3, §6.5)

**Files:**
- Create: `packages/geo-core/src/hierarchy.ts`, `packages/geo-core/src/api-contract.ts`
- Test: `packages/geo-core/test/hierarchy.test.ts`

**Interfaces:**
- Produces:
  - `tileOf(cell: string): string` : la parente de résolution `TILE_RES`.
  - `type LodEntry = { res: number; cell: string; r3: string }`
  - `lodEntries(cell: string): LodEntry[]` : une entrée par résolution de `LOD_RESOLUTIONS`, dans cet ordre, avec `r3 = cellToParent(cell, LOD_FILTER_RES)`.
  - `isValidTileId(id: string): boolean`
  - Dans `api-contract.ts`, les types de la spec §6.5, copiés tels quels : `ZoneLevel = 'country' | 'region' | 'city' | 'district'`, `Zone`, `TileResponse`, `MetaResponse = { datasetVersion: number; worldStreetCellEstimate: number; countries: { code: string; osmTimestamp: string; importedAt: string }[] }`, `ApiError = { error: { code: string; message: string } }`.

- [ ] **Step 1 : écrire les tests qui échouent**

```ts
const cell = latLngToCell(48.8634, 2.3601, 12);
it('tileOf returns the resolution-7 parent', () => {
  expect(tileOf(cell)).toBe(cellToParent(cell, 7));
});
it('lodEntries returns resolutions 10, 8, 6, 4 with their res-3 parent', () => {
  const entries = lodEntries(cell);
  expect(entries.map((e) => e.res)).toEqual([10, 8, 6, 4]);
  for (const e of entries) {
    expect(e.cell).toBe(cellToParent(cell, e.res));
    expect(e.r3).toBe(cellToParent(cell, 3));
  }
});
it('isValidTileId accepts only canonical lowercase resolution-7 cells', () => {
  const tile = cellToParent(cell, 7);
  expect(isValidTileId(tile)).toBe(true);
  expect(isValidTileId(cell)).toBe(false);
  expect(isValidTileId(tile.toUpperCase())).toBe(false);
  expect(isValidTileId('abc')).toBe(false);
  expect(isValidTileId('')).toBe(false);
});
```

- [ ] **Step 2 : lancer les tests pour vérifier qu'ils échouent**

Run: `pnpm --filter @fog/geo-core test hierarchy`
Expected: FAIL

- [ ] **Step 3 : implémenter.** `isValidTileId` combine `/^[0-9a-f]{15}$/`, `isValidCell` et `getResolution(id) === TILE_RES`. Ajouter `api-contract.ts` (types uniquement) et réexporter le tout depuis `index.ts`.

- [ ] **Step 4 : lancer les tests et la vérification des types**

Run: `pnpm --filter @fog/geo-core test && pnpm --filter @fog/geo-core typecheck`
Expected: PASS

- [ ] **Step 5 : commit**

```bash
git add -A && git commit -m "feat(geo-core): add cell hierarchy helpers and API contract types"
```

---

### Task 6 : calcul des pourcentages (spec §5.4)

**Files:**
- Create: `packages/geo-core/src/progress.ts`
- Test: `packages/geo-core/test/progress.test.ts`

**Interfaces:**
- Consumes: `Zone` (Task 5).
- Produces:
  - `type ZoneProgress = { zoneId: string; level: ZoneLevel; name: string; discovered: number; total: number; ratio: number }`
  - `type WorldProgress = { discovered: number; estimate: number | null; ratio: number | null }`
  - `computeProgress(countsByZone: Record<string, number>, zones: Zone[], worldEstimate: number | null): { zones: Record<string, ZoneProgress>; world: WorldProgress }`. `countsByZone` = nombre de cellules de rue découvertes par zone la plus fine. Chaque zone cumule ses descendantes. Le numérateur du monde est la somme de tous les comptes, y compris ceux de zones inconnues.
  - `zoneChain(zoneId: string, zones: Zone[]): Zone[]` : de la zone la plus fine jusqu'au pays. S'arrête sur un parent inconnu ou un cycle.

- [ ] **Step 1 : écrire les tests qui échouent**

```ts
const z = (id: string, level: ZoneLevel, parentId: string | null, streetCellCount: number): Zone =>
  ({ id, osmId: 0, level, name: id, parentId, countryCode: 'fr', streetCellCount });
const zones = [
  z('fr', 'country', null, 1000), z('idf', 'region', 'fr', 400),
  z('paris', 'city', 'idf', 200), z('p3', 'district', 'paris', 20),
  z('versailles', 'city', 'idf', 50), z('empty', 'city', 'idf', 0),
];

it('rolls discovered counts up the hierarchy', () => {
  const { zones: r } = computeProgress({ p3: 10, versailles: 5 }, zones, null);
  expect(r.p3!.ratio).toBe(0.5);
  expect(r.paris!.ratio).toBe(0.05);
  expect(r.idf!.ratio).toBe(0.0375);
  expect(r.fr).toMatchObject({ discovered: 15, total: 1000, ratio: 0.015 });
});
it('never returns NaN for a zone with zero street cells', () => {
  expect(computeProgress({}, zones, null).zones.empty!.ratio).toBe(0);
});
it('computes the world ratio from the estimate, or null without one', () => {
  expect(computeProgress({ p3: 10, unknown: 5 }, zones, 1_000_000).world)
    .toEqual({ discovered: 15, estimate: 1_000_000, ratio: 0.000015 });
  expect(computeProgress({ p3: 10 }, zones, null).world.ratio).toBeNull();
  expect(computeProgress({ p3: 10 }, zones, 0).world.ratio).toBeNull();
});
it('zoneChain walks from the finest zone to the country and survives cycles', () => {
  expect(zoneChain('p3', zones).map((x) => x.id)).toEqual(['p3', 'paris', 'idf', 'fr']);
  const cyclic = [z('a', 'city', 'b', 1), z('b', 'region', 'a', 1)];
  expect(zoneChain('a', cyclic).map((x) => x.id)).toEqual(['a', 'b']);
});
```

- [ ] **Step 2 : lancer les tests pour vérifier qu'ils échouent**

Run: `pnpm --filter @fog/geo-core test progress`
Expected: FAIL

- [ ] **Step 3 : implémenter `computeProgress` et `zoneChain`.** Pour chaque entrée de `countsByZone`, ajouter le compte à toutes les zones de `zoneChain(zoneId)`. `ratio = total > 0 ? discovered / total : 0`. Le ratio du monde vaut `null` si `!worldEstimate`.

- [ ] **Step 4 : lancer les tests pour vérifier qu'ils passent**

Run: `pnpm --filter @fog/geo-core test`
Expected: PASS

- [ ] **Step 5 : commit**

```bash
git add -A && git commit -m "feat(geo-core): compute exploration progress per zone and world"
```

---

### Task 7 : conversion des géométries en cellules de rue (spec §6.1, étape 5)

**Files:**
- Create: `packages/geo-core/src/geometry.ts`
- Test: `packages/geo-core/test/geometry.test.ts`

**Interfaces:**
- Consumes: `haversineMeters` (Task 3).
- Produces:
  - `type LngLat = [lng: number, lat: number]`
  - `lineToCells(coords: LngLat[]): Set<string>` : densifie chaque segment avec un pas ≤ `DENSIFY_STEP_M`, convertit chaque point à la résolution `DISCOVERY_RES` et dédoublonne.
  - `areaToCells(polygon: GeoJSON.Polygon | GeoJSON.MultiPolygon): Set<string>` = `polygonToCells(..., DISCOVERY_RES, true)` ∪ `lineToCells` de chaque anneau. Ajouter `@types/geojson` en dépendance de développement.

- [ ] **Step 1 : écrire les tests qui échouent**

Créer d'abord `packages/geo-core/test/helpers.ts`, qui exporte `isConnected(cells: Set<string>): boolean`. Cette fonction fait un parcours en largeur depuis une cellule quelconque, en suivant `areNeighborCells`, et renvoie `true` si toutes les cellules sont atteintes (et aussi pour un ensemble vide).

```ts
import { isConnected } from './helpers';
const line: LngLat[] = [[2.3601, 48.8634], [2.3601, 48.8643]]; // ~100 m

it('covers both ends of a line with a connected set of cells', () => {
  const cells = lineToCells(line);
  expect(cells.has(latLngToCell(48.8634, 2.3601, 12))).toBe(true);
  expect(cells.has(latLngToCell(48.8643, 2.3601, 12))).toBe(true);
  expect(isConnected(cells)).toBe(true);
});
it('handles degenerate lines', () => {
  expect(lineToCells([[2.3601, 48.8634]]).size).toBe(1);
  expect(lineToCells([line[0]!, line[0]!, line[1]!])).toEqual(lineToCells(line));
  expect(lineToCells([]).size).toBe(0);
});
it('fills an area and includes its boundary', () => {
  const square: GeoJSON.Polygon = { type: 'Polygon', coordinates: [[
    [2.3600, 48.8630], [2.3614, 48.8630], [2.3614, 48.8639], [2.3600, 48.8639], [2.3600, 48.8630],
  ]] }; // ~100 m × 100 m
  const cells = areaToCells(square);
  expect(cells.has(latLngToCell(48.86345, 2.3607, 12))).toBe(true); // centre
  expect(cells.has(latLngToCell(48.8630, 2.3600, 12))).toBe(true);  // coin
  expect(isConnected(cells)).toBe(true);
});
it('returns an empty set for an invalid ring', () => {
  expect(areaToCells({ type: 'Polygon', coordinates: [[[2.36, 48.86], [2.361, 48.86]]] }).size).toBe(0);
});
```

- [ ] **Step 2 : lancer les tests pour vérifier qu'ils échouent**

Run: `pnpm --filter @fog/geo-core test geometry`
Expected: FAIL

- [ ] **Step 3 : implémenter.** Densification : pour un segment de longueur `d`, prendre `n = max(1, ceil(d / DENSIFY_STEP_M))` points interpolés linéairement en lat/lng (exact à moins de 1 m sur ces distances). Un anneau de moins de 4 positions est ignoré, sans exception levée.

- [ ] **Step 4 : lancer les tests pour vérifier qu'ils passent**

Run: `pnpm --filter @fog/geo-core test`
Expected: PASS

- [ ] **Step 5 : commit**

```bash
git add -A && git commit -m "feat(geo-core): convert OSM lines and areas to street cells"
```

---

### Task 8 : lecture des fichiers GPX et scénarios de référence (spec §10)

**Files:**
- Create: `packages/geo-core/src/gpx.ts`
- Create: `packages/geo-core/test/fixtures/{walk,tunnel,flight,jump}.gpx`
- Test: `packages/geo-core/test/gpx.test.ts`, `packages/geo-core/test/scenarios.test.ts`

**Interfaces:**
- Consumes: `processBatch` (Task 4), `GeoPoint` (Task 3).
- Produces: `parseGpx(xml: string, defaultAccuracy?: number): GeoPoint[]`, avec `defaultAccuracy = 10`. Lit chaque `<trkpt lat lon>` et son `<time>` ISO. Les points sans `<time>` valide sont ignorés. Implémenté sans dépendance (expressions régulières), pour fonctionner aussi dans React Native. Le plan 3 l'utilise dans `ReplayLocationSource`.

**Contenu des fichiers de test** (le GPX ne porte pas de précision, on utilise donc `defaultAccuracy`) :
- `walk.gpx` : 20 points, `lng = 2.3601`, `lat = 48.8600 + i × 0.0001`, un toutes les 8 s à partir de `2026-10-16T10:00:00Z`.
- `tunnel.gpx` : `(48.8600, 2.3601)` à 10:00:00, puis `(48.8735, 2.3601)` (~1,5 km) à 10:07:00.
- `flight.gpx` : `(49.0097, 2.5479)` à 10:00:00, `(49.1447, 2.5479)` (~15 km) à 10:01:00 (900 km/h), puis `(35.7720, 140.3929)` (Narita) le `2026-10-18T10:00:00Z`. On attend 48 h et non 24 h : Paris-Tokyo représente ~9 700 km, et en 24 h la vitesse apparente dépasserait 400 km/h.
- `jump.gpx` : comme `walk.gpx`, mais le 10e point est remplacé par `(48.8780, 2.3601)` (~2 km plus loin), au même horodatage.

- [ ] **Step 1 : écrire les tests qui échouent**

```ts
// gpx.test.ts
it('parses trackpoints and skips points without time', () => {
  const xml = `<gpx><trk><trkseg>
    <trkpt lat="48.86" lon="2.36"><time>2026-10-16T10:00:00Z</time></trkpt>
    <trkpt lat="48.87" lon="2.37"></trkpt>
  </trkseg></trk></gpx>`;
  expect(parseGpx(xml)).toEqual([{ lat: 48.86, lng: 2.36, accuracy: 10, timestamp: Date.parse('2026-10-16T10:00:00Z') }]);
});

// scenarios.test.ts
const run = (name: string) => processBatch(parseGpx(readFileSync(`${__dirname}/fixtures/${name}.gpx`, 'utf8')), null);

it('walk: a connected trail covering every point', () => {
  const { cells } = run('walk');
  expect(isConnected(new Set(cells.keys()))).toBe(true);
  for (let i = 0; i < 20; i++) expect(cells.has(latLngToCell(48.86 + i * 0.0001, 2.3601, 12))).toBe(true);
});
it('tunnel: no straight line across the gap', () => {
  expect(run('tunnel').cells.size).toBe(14);
});
it('flight: nothing revealed in the air, landing still counted', () => {
  const { cells } = run('flight');
  expect(cells.size).toBe(14);
  expect(cells.has(latLngToCell(49.1447, 2.5479, 12))).toBe(false);
  expect(cells.has(latLngToCell(35.772, 140.3929, 12))).toBe(true);
});
it('jump: the outlier point is ignored', () => {
  expect(run('jump').cells.has(latLngToCell(48.878, 2.3601, 12))).toBe(false);
});
```

`isConnected` est importé depuis `./helpers` (Task 7).

- [ ] **Step 2 : lancer les tests pour vérifier qu'ils échouent**

Run: `pnpm --filter @fog/geo-core test gpx scenarios`
Expected: FAIL

- [ ] **Step 3 : implémenter `parseGpx` et écrire les 4 fichiers de test** avec les coordonnées ci-dessus.

- [ ] **Step 4 : lancer toute la suite et les vérifications**

Run: `pnpm test && pnpm lint && pnpm typecheck`
Expected: tout passe.

- [ ] **Step 5 : commit**

```bash
git add -A && git commit -m "feat(geo-core): parse GPX traces and add reference scenarios"
```
