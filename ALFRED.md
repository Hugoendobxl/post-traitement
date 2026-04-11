# ALFRED.md — post-traitement

> Documentation vivante à jour au 2026-04-11. Ce fichier est lu par Alfred au démarrage de toute session Claude Code sur ce module. Il doit être mis à jour à chaque modification significative du code.

## 1. Rôle du module

Frontend pour les assistantes : envoi d'emails post-opératoires aux patients après leur traitement endodontique. L'email contient un lien vers les avis Google pour solliciter un avis positif. Les assistantes saisissent au desk les patients du jour, choisissent la langue, puis déclenchent les envois en masse via le bouton "Envoyer". Affiche aussi les KPIs Brevo et les avis Google récents.

## 2. URLs et identifiants

- **Frontend** : https://post-traitement.vercel.app
- **Backend (partagé)** : https://cabinet-finance-api-production.up.railway.app — ⚠️ **PARTAGÉ avec le module Finance**
- **Dépôt GitHub** : Hugoendobxl/post-traitement
- **Projet Vercel** : `post-traitement`
- **Projet Railway (du backend partagé)** : `modest-unity`

## 3. Stack technique

- **Runtime** : navigateur
- **Framework** : **aucun** — fichier HTML autonome unique (`index.html`, ~1149 lignes, ~151 KB) avec HTML + CSS + JS inline (vanilla, syntaxe ES5 `var`)
- **Build** : aucun
- **Hébergement** : Vercel (déploiement direct du fichier statique)
- **Authentification** : ⚠️ **AUCUNE** au niveau du module (le frontend est accessible sans PIN). La sécurité repose sur le fait que personne ne connaît l'URL hors du cabinet.

## 4. Variables d'environnement

Aucune. Toutes les URLs backend sont codées en dur dans `index.html` :

```js
var PT_API = 'https://cabinet-finance-api-production.up.railway.app/api/post-traitement';
var KPI_API = 'https://cabinet-finance-api-production.up.railway.app/api/kpi-stats';
var REVIEWS_API = 'https://cabinet-finance-api-production.up.railway.app/api/google-reviews';
```

## 5. Structure du projet

```
post-traitement/
├── index.html         # Tout le frontend (1149 lignes, 151 KB)
└── index.html.bak     # Backup d'une version précédente (128 KB) — à investiguer / nettoyer
```

⚠️ **Pas de `package.json`, pas de `vercel.json`, pas de `.gitignore`**. C'est un dépôt minimaliste — Vercel sert simplement `index.html`.

## 6. Points d'entrée API (consommés)

Toutes les routes consommées sont dans `cabinet-finance-api` (préfixe `/api`) :

| Route consommée | Méthode | Usage |
|---|---|---|
| `/api/send-post-treatment` | POST | Envoi d'un email Brevo (body : `{email, templateId, params}`) |
| `/api/post-traitement/patients` | GET / PUT | Liste partagée du jour (synchronisée entre tous les postes du cabinet) |
| `/api/post-traitement/kpi-sent` | GET | Compteur d'emails envoyés |
| `/api/post-traitement/kpi-sent/increment` | POST | Incrément du compteur |
| `/api/kpi-stats` | GET | KPIs Brevo (clics) + KPIs Google Places (note moyenne) |
| `/api/google-reviews` | GET | Liste des avis Google |
| `/api/google-reviews/sync` | POST | Force la resynchronisation des avis |
| `/api/google-reviews/ai-draft` | POST | Génération IA d'une réponse à un avis (Anthropic) |
| `/api/google-reviews/:id/reply` | POST | Publication de la réponse |

## 7. Schéma de base de données

N/A (frontend pur). La table `post_traitement_data` est gérée par `cabinet-finance-api` (créée à la volée par `post-treatment.js`).

## 8. Écrans / Pages

Une seule page (single HTML), avec plusieurs sections :

- **Sélecteur de langue** : `Français / Nederlands / English` (boutons `tb` lignes 292-294)
- **Saisie d'un patient** : nom, email, langue, dent, endodontiste — bouton "Envoyer" qui appelle `/api/send-post-treatment` puis incrémente le compteur
- **Liste des patients du jour partagée** : récupérée via `/api/post-traitement/patients`, envoi en masse possible
- **KPIs Brevo + Google** : affichage des clics email et de la note moyenne du cabinet
- **Section Google Reviews** : liste des avis avec bouton "Brouillon IA" + "Répondre"

## 9. Règles métier critiques

- **Mapping `templateId` → langue** (lignes 539 et 866, **dupliqué** !) :
  - `fr` → templateId Brevo **3**
  - `nl` → templateId Brevo **5**
  - `en` → templateId Brevo **6**
- **État global minimal** : `var state = { langue: 'fr' };` (ligne 427) — pas de framework, pas de store
- **Auto-wipe quotidien** de la liste des patients : géré côté backend (`post-treatment.js` ligne 45 — supprime la donnée si elle date d'un jour précédent)
- **Compteur KPI envoyés persisté en base** via `cabinet-finance-api` (clé `kpi-sent` dans `post_traitement_data`)
- **Style sombre** : couleurs `#CE93D8` (lavande) sur fond `#0A0614` (très sombre, presque noir) — décor "particules" type page de garde

## 10. Intégrations externes

| Service | Rôle | Variables d'env |
|---|---|---|
| `cabinet-finance-api` (Railway) | Backend complet (Brevo + KV store + Google Reviews) | URLs en dur |

Aucune intégration externe directe — tout passe par le backend partagé.

## 11. Dépendances avec d'autres modules de l'écosystème

### 🚨 POINT CRITIQUE — BREVO PARTAGÉ AVEC LE MODULE FINANCE

Ce module **n'a pas son propre backend**. Il consomme **`cabinet-finance-api`** pour :
1. Envoyer les emails post-op (`POST /api/send-post-treatment` → `mailer.js → sendPostTreatment` → API Brevo)
2. Synchroniser la liste partagée des patients du jour entre tous les postes du cabinet
3. Afficher les KPIs et les avis Google

**Conséquence** : la **même clé `BREVO_API_KEY`** est utilisée par :
- L'OTP de login d'Hugo pour le module Finance
- Les emails post-op envoyés par les assistantes

Si on casse Brevo dans `cabinet-finance-api`, on casse **les deux modules en même temps**. Voir le détail dans `cabinet-finance-api/ALFRED.md` §11.

### Autres dépendances

**Appelle** :
- `cabinet-finance-api` (toutes les routes `/api/*`)

**Est appelé par** :
- `endodontie-launcher-v2` (tuile "Email Post-Traitement" — ouverture dans nouvel onglet)

## 12. Déploiement

- **Déclenchement** : push sur `main` → auto-deploy Vercel
- **Pas de build** : Vercel sert directement `index.html`
- **Branche de production** : `main`

## 13. Fichiers sensibles à ne pas modifier sans validation

- `index.html` — c'est tout le module ; toute modification doit être testée

## 14. Commandes utiles

```bash
# Pas de commande de build/dev — fichier statique
# Pour tester en local
open index.html
# ou
python3 -m http.server 8000
```

## 15. Historique des décisions importantes

- **Pas de framework, pas de package.json** : choix volontaire de garder le module ultra-simple. La logique est minimale (formulaire + appels API), pas besoin de React/Vue.
- **Pas d'auth dédiée** : décision pragmatique parce que le module est déjà derrière le launcher (qui a son propre PIN). La sécurité par obscurité peut être renforcée plus tard si besoin.
- **Backend partagé avec Finance** : décision historique d'éviter de monter un nouveau backend juste pour 4 endpoints. Pratique mais crée le couplage Brevo critique (§11).
- **Mapping `templateId` dupliqué** (lignes 539 et 866) : à factoriser si on retravaille le code.

## 16. TODO et points d'attention

- **Aucun commentaire `TODO/FIXME`** détecté.
- **`index.html.bak`** (128 KB) à la racine du repo — backup d'une version précédente. À supprimer après validation que la version actuelle marche, ou à archiver proprement.
- **Mapping `templateId → langue` dupliqué** entre lignes 539 et 866 : à factoriser en une seule variable globale
- **Aucune authentification** — risque si l'URL fuite. À discuter pour la suite.
- **Pas de tests automatisés**

---

**Dernière mise à jour** : 2026-04-11
**Mis à jour par** : Claude Code (session naissance du workspace, Alfred)
