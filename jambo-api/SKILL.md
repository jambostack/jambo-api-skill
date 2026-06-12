---
name: jambo-api
description: Use when reading from or writing to a Jambo headless CMS API (collections, entries, fields, end-user auth) in ANY project. Covers endpoints, request/response shapes, token abilities, pagination/limits, error codes, and the flat-vs-nested field gotcha. Triggers whenever code calls a Jambo API host, hydrates a page from a Jambo CMS, or runs collection/field/entry seed scripts.
---

# Jambo API — guide pratique (vérifié sur la source Symfony)

Jambo est un CMS headless (Symfony/PHP). Ce guide reflète le **code source** (controllers
`ContentController`, `CollectionController`, service `ApiTokenChecker`, entité `ApiToken`),
pas un seul projet. Remplacer les placeholders :

| Placeholder       | Signification                                   |
|-------------------|-------------------------------------------------|
| `{API_BASE}`      | Hôte de l'API, ex. `https://api.exemple.com`    |
| `{PROJECT_UUID}`  | UUID du projet Jambo (format UUID)              |
| `{collection}`    | Slug d'une collection                           |
| `{uuid}`          | UUID d'une entrée                               |

> **Secrets :** ne jamais committer ni exposer un token au navigateur. Le charger depuis
> l'environnement (`JAMBO_TOKEN`, `JAMBO_ADMIN_TOKEN`) ou un fichier hors dépôt.

> ⚠️ **Vérifie d'abord pour TON projet** (ces réglages varient et changent tout) :
> 1. **`publicApi` activé ?** Sinon la lecture publique renvoie **403** (il faudra un token).
> 2. **Abilities de ton token ?** `read` ne suffit pas pour écrire (voir §1).
> 3. **Version de l'API** : confirme une route par un appel réel avant de scripter en masse.

---

## 1. Concepts & abilities de token

- **Projet** : espace isolé identifié par `{PROJECT_UUID}`. `publicApi` (booléen) ouvre la
  lecture **anonyme** du contenu publié.
- **Collection** : type de contenu. `isSingleton: true` ⇒ une seule entrée (créer une 2ᵉ → **409**).
- **Champ** : `{ slug, type, isRequired, options }`.
- **Entrée** : `uuid`, `locale`, `status` (`draft`|`published`) + valeurs de champs.
- **ApiToken** (`Authorization: Bearer <token>`) : porte un tableau `abilities`
  (défaut `['read']`). Vérification stricte `in_array(ability, abilities)` :
  - **lecture de schéma** : n'importe quel token **valide** (les abilities ne sont PAS testées).
  - **créer / modifier une entrée** (voie publique) : ability **`create`**.
  - **supprimer une entrée** : ability **`delete`**.
  - Token absent/invalide/expiré sur un endpoint protégé → **401**.

Conventions : slug de **collection** sans underscore (`formationscores`) ; slugs de **champ**
libres (`cta_label`, `poleSlug`). CORS `*` (fetch navigateur OK). En-têtes d'écriture :
`Authorization: Bearer`, `Content-Type: application/json`, `Accept: application/json`.

---

## 2. Lecture de contenu — voie publique (anonyme si `publicApi`)

```bash
GET {API_BASE}/api/{PROJECT_UUID}/{collection}            # liste
GET {API_BASE}/api/{PROJECT_UUID}/{collection}/{uuid}     # une entrée
```
Query : `locale` (défaut = locale par défaut du projet), `page` (défaut 1),
`per_page` (défaut **15**, **plafonné à 100**), `status` (défaut **`published`** ; seuls
`draft`/`published` acceptés).

- Si `publicApi=false` → **403** « Public API access is disabled ». Projet introuvable → 404.
- **Liste** : `{ "data": [ … ], "meta": { total, page, per_page, pages } }`.
- **Entrée seule** : l'objet entrée **directement** (pas d'enveloppe `data`).
- **Les valeurs de champ sont des clés au niveau racine de l'entrée** (`heading`, `nom`…),
  à côté de `id/uuid/locale/status/collection/...`. Pas de sous-objet `fields`. Champs vides ⇒ souvent absents.

```js
// Hydratation client typique (lecture publique, sans token, avec fallback)
const r = await fetch(`${API_BASE}/api/${PID}/temoignages?locale=fr&per_page=100`);
const items = r.ok ? (await r.json()).data : [];
if (items.length) render(items.map(t => ({ nom: t.nom, texte: t.texte, note: t.note })));
// si !r.ok ou items vide → ne rien toucher (garder le HTML existant)
```

---

## 3. Schéma d'une collection (token valide requis)

```bash
GET {API_BASE}/public-api/collections            # liste (le projet vient du TOKEN)
GET {API_BASE}/public-api/collections/{collection}
```
Renvoie `{ name, slug, description, isSingleton, fields:[{name, slug, type, isRequired, options}] }`.
Token **valide** suffit (n'importe quelle ability). Pas de token → 401. *(Un 401 avec un token
de lecture « correct » signifie le plus souvent un token **périmé/régénéré**, pas un manque de droit.)*

---

## 4. Écriture de contenu — VOIE PROGRAMMATIQUE RECOMMANDÉE (à plat, ApiToken)

C'est la voie à utiliser par défaut pour scripts/sites. Token avec ability `create`/`delete`.

```bash
POST   {API_BASE}/api/{PROJECT_UUID}/{collection}          # créer    → can('create'), 201
PUT    {API_BASE}/api/{PROJECT_UUID}/{collection}/{uuid}   # modifier → can('create'), 200
PATCH  {API_BASE}/api/{PROJECT_UUID}/{collection}/{uuid}
DELETE {API_BASE}/api/{PROJECT_UUID}/{collection}/{uuid}   # supprimer → can('delete'), 204
```

**Corps = valeurs de champ À PLAT**, au niveau racine, à côté de `locale`/`status` :
```json
{ "locale": "fr", "status": "published", "<fieldSlug>": "valeur", "<autre>": 123 }
```

> 🔴 **Deux pièges majeurs, vérifiés dans le code :**
> 1. **À plat, pas imbriqué.** Le serveur lit `data[field.slug]`. Si tu mets `"fields": {…}`,
>    l'entrée est créée **mais les champs restent vides** — et tu reçois quand même **201**.
>    ⇒ Toujours **re-GET** l'entrée après écriture pour vérifier les valeurs.
> 2. **`status` défaut = `draft`.** Sans `"status":"published"`, l'entrée n'apparaît PAS en
>    lecture publique (qui ne renvoie que `published` par défaut).

Réponse de création/màj = l'objet entrée **à la racine** (`uuid` au top niveau). Le projet de l'URL
doit correspondre au projet du token (sinon **403**).

---

## 5. Voie « studio / dashboard » — imbriquée (`/api/projects/…`)

Utilisée par l'interface d'admin (authentifiée par **session utilisateur**, pas par ApiToken),
et par certains scripts de structuration. Elle gère la **structure** (collections, champs) :

```bash
POST   {API_BASE}/api/projects/{PROJECT_UUID}/collections                 # {name, slug, is_singleton, order}
POST   {API_BASE}/api/projects/{PROJECT_UUID}/collections/{collection}/fields   # {name, type, slug?, is_required?} — name+type requis (sinon 422)
POST   {API_BASE}/api/projects/{PROJECT_UUID}/collections/{collection}/entries  # entrée
DELETE {API_BASE}/api/projects/{PROJECT_UUID}/collections/{collection}/entries/{uuid}/force-delete  # hard delete
```

> ⚠️ Ici le corps d'entrée est **IMBRIQUÉ** : `{ locale, status, "fields": { … } }` (le serveur
> lit `data['fields']`). C'est l'inverse de §4. Ne pas confondre les deux voies.
> Locale hors des locales du projet → 422 ; 2ᵉ entrée d'un singleton → 409.

**Quelle voie choisir ?** Programmatique/site → **§4** (publique, à plat, ApiToken).
Création de collections/champs ou outillage dashboard → **§5** (imbriqué).

---

## 6. Types de champs & valeurs spéciales

Types à stockage particulier (le reste → texte) :
`number`/`decimal` (numérique) · `boolean`/`checkbox` (bool) · `date`/`datetime`/`time` ·
`json`/`array`/`repeater` (JSON) · `media`/`relation`/`enumeration`.

> **`media`/`relation` = tableaux d'UUID** appartenant au projet (validés ; un UUID d'un autre
> projet est **silencieusement ignoré** — protection IDOR). On ne met donc PAS une URL d'image
> brute dans un champ `media` : il faut des UUID de fichiers, uploadés via l'API fichiers
> `{API_BASE}/api/{PROJECT_UUID}/files` (lecture aussi soumise à `publicApi`).

---

## 7. Auth end-user (JWT) — ne donne PAS l'écriture de contenu

```bash
POST {API_BASE}/api/{PROJECT_UUID}/auth/{register,login,refresh,logout,forgot-password,reset-password}
GET/PATCH {API_BASE}/api/{PROJECT_UUID}/auth/me
```
`login` → `{ data: { access_token, user: { custom_fields } } }`. Le JWT authentifie un
**utilisateur d'application** ; il **ne crée pas** de contenu (réservé à un ApiToken `create`).

---

## 8. Pattern « site statique + proxy » (sécurité)

Un site statique ne peut pas cacher de secret. Pattern sûr :
- **Lectures** de contenu : directes/anonymes (§2) — aucun token côté client.
- **Schéma + écritures** : relayées par un **proxy serveur** détenant le token (vérifie une
  session avant de relayer). Le token ne quitte jamais le serveur.
- Config secrète **hors webroot** mais dans un emplacement lisible par le runtime (PHP-FPM/HestiaCP :
  attention à `open_basedir` — un dossier `private/` du domaine, sinon erreur 500).

---

## 9. Client de référence (point de départ — évite de réécrire un wrapper)

```js
// jambo.mjs — lecture publique + écriture (voie §4). Token via env, jamais en dur.
const API = process.env.JAMBO_API_BASE;          // https://api.exemple.com
const PID = process.env.JAMBO_PROJECT_UUID;
const TOKEN = process.env.JAMBO_TOKEN;            // ability create/delete pour écrire

const read = (c, { locale = 'fr', perPage = 100, status = 'published' } = {}) =>
  fetch(`${API}/api/${PID}/${c}?locale=${locale}&per_page=${Math.min(perPage,100)}&status=${status}`)
    .then(r => r.ok ? r.json().then(j => j.data ?? []) : Promise.reject(r.status));

const write = (c, fields, { uuid, locale = 'fr', status = 'published' } = {}) => {
  const body = JSON.stringify({ locale, status, ...fields });   // ← À PLAT
  const url = uuid ? `${API}/api/${PID}/${c}/${uuid}` : `${API}/api/${PID}/${c}`;
  return fetch(url, { method: uuid ? 'PUT' : 'POST',
    headers: { Authorization: `Bearer ${TOKEN}`, 'Content-Type': 'application/json', Accept: 'application/json' },
    body }).then(async r => (r.ok ? r.json() : Promise.reject(`${r.status} ${await r.text()}`)));
};

const remove = (c, uuid) => fetch(`${API}/api/${PID}/${c}/${uuid}`,
  { method: 'DELETE', headers: { Authorization: `Bearer ${TOKEN}` } }).then(r => r.status);

export { read, write, remove };
```

---

## 10. Codes & erreurs

`201` créé · `200` ok · `204` supprimé · `401` token manquant/invalide/expiré ·
`403` `publicApi` désactivé / projet ≠ token / ability manquante · `404` projet/collection/entrée ·
`409` 2ᵉ entrée d'un singleton · `422` validation (locale non activée, champ sans name/type).
Corps d'erreur : `{ "error": "…" }`.

---

## 11. Checklist anti-bug

- [ ] Écriture **voie §4** → champs **à plat** + `status:"published"` (défaut = draft) + re-GET de vérif.
- [ ] Voie **§5** (studio) → champs **imbriqués** sous `fields`. Ne pas croiser §4/§5.
- [ ] `per_page` ≤ **100** (au-delà, plafonné).
- [ ] Lecture publique suppose `publicApi=true` (sinon 403) ; écriture suppose ability `create`/`delete`.
- [ ] `media`/`relation` = **UUID** du projet (pas d'URL brute).
- [ ] Slug de collection **sans underscore** ; token jamais committé/exposé au navigateur (§8).
- [ ] Nettoyer toute entrée de test après une sonde sur un environnement réel.
