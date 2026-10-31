# 🗺️ ROADMAP — FripShop API (Django REST Framework)

> Document de référence du projet. Chaque phase se termine par un **livrable** et respecte la **Definition of Done (DoD)** commune.
> Ce document couvre le **backend**. Le web (React) et le mobile ont leurs propres repos (voir la section « Pistes front »).

---

## 0. Vision & périmètre

| Élément | Valeur |
|---|---|
| Produit | **FripShop** : marketplace de friperie (fripiers pros vérifiés) — ADR 0002 |
| Catégories | Vêtements, chaussures, sacs et accessoires, maison |
| Marché | Tout le Cameroun ; vendeurs recrutés d'abord à Douala (et Yaoundé) |
| Devise | XAF (FCFA) |
| Langue | Français (anglais en option) |
| Architecture | API REST DRF + JWT — ADR 0003 |
| Clients | Site web React, app mobile |
| Paiements | Mobile Money (MTN MoMo, Orange Money) via un agrégateur **capable de faire des reversements**, paiement à la livraison (zones limitées) |

### Jalons

| Version | Contenu | Phases |
|---|---|---|
| **v0.1 — Socle** | API qui tourne, CI verte, comptes et JWT | 0 → 2 |
| **v0.3 — Vendeurs** | Inscription des vendeurs, publication d'annonces, catalogue public. **Mis en ligne tôt** pour que les fripiers remplissent le catalogue avant l'ouverture aux acheteurs. | 3 → 4 |
| **v1.0 — MVP** | Achat complet payé, livré, vendeur reversé, litiges de base, notifications, durci, en production | 5 → 12 + 14 + 15 |
| **v1.x — Croissance** | Avis, promos, favoris, offres, recommandations | 13 |

---

## Stack technique

| Couche | Outil | Rôle |
|---|---|---|
| Backend | Django 5.x + Django REST Framework | API REST |
| Auth | djangorestframework-simplejwt | JWT (rotation + blacklist) |
| Doc API | drf-spectacular | OpenAPI 3, Swagger UI |
| Filtres / CORS | django-filter, django-cors-headers | Recherche, accès depuis React |
| Base de données | PostgreSQL 16 | Données, transactions, recherche full-text |
| Cache / broker | Redis 7 | Cache, throttling, file de tâches |
| Tâches async | Celery + Celery Beat | Emails, push, webhooks, expirations, reversements |
| Images | Pillow + stockage S3-compatible | Photos des annonces (compression, miniatures) |
| Qualité | ruff, black, mypy (django-stubs, drf-stubs), pre-commit | Standards |
| Tests | pytest-django, factory_boy, coverage | ≥ 80 % |
| Environnement | Docker, docker-compose, django-environ | |
| CI/CD | GitHub Actions | Lint + tests + validation du schéma OpenAPI |
| Production | Gunicorn + Nginx, Sentry | |
| Front (hors repo) | React (web), app mobile (techno à décider) | Consomment l'API |

---

## ✅ Definition of Done (commune à toute feature)

- [ ] Code conforme à `CLAUDE.md`
- [ ] Tests verts, y compris **permissions** (401 non authentifié, **404** sur l'objet d'un autre utilisateur)
- [ ] `ruff`, `black`, `mypy` sans erreur
- [ ] Endpoint documenté (`@extend_schema`), `spectacular --validate` sans warning
- [ ] Migrations générées et versionnées
- [ ] CI verte
- [ ] `PROGRESS.md` mis à jour

---

## Phase 0 — Cadrage *(aucun code)*

- [x] Type de plateforme : marketplace (ADR 0002)
- [x] Vision produit (`VISION.md`)
- [x] Architecture API DRF + JWT (ADR 0003)
- [x] Rôles : visiteur, client, vendeur, admin (`USER_STORIES.md` § 2)
- [x] User stories par rôle + MoSCoW (`USER_STORIES.md`)
- [x] ERD complet (Order → SellerOrder → OrderItem, Listing, Seller, registre des soldes, Payout) (`ERD.md`)
- [x] Cycle de vie des annonces et machine à états de `SellerOrder` (`ERD.md` § 4)
- [x] Choix de l'agrégateur : **Campay** (ADR 0004)
- [x] Règles métier : commission, frais, retours, pénalités, KYC, livraison (`USER_STORIES.md` § 8)
- [x] Première version de la liste des endpoints (contrat API pour le front) (`API.md`)

**Livrables :** `VISION.md`, `USER_STORIES.md`, `ERD.md`, `API.md` (liste des endpoints), ADR.

---

## Phase 1 — Fondations

- [ ] Repo, `.gitignore`, branches
- [ ] Structure (voir `CLAUDE.md`), settings découpés, django-environ
- [ ] Docker : web, db, redis, celery, celery-beat
- [ ] Config DRF : auth JWT par défaut, `IsAuthenticated` par défaut, pagination, throttling, versionnement `/api/v1/`
- [ ] drf-spectacular (schéma + Swagger en dev), CORS
- [ ] App `core` : `TimeStampedModel`, `public_id` UUID, `ApplicationError` + **gestionnaire d'erreurs global**
- [ ] App `accounts` : **Custom User** avant la 1ʳᵉ migration
- [ ] Endpoint `/api/v1/health/`
- [ ] pytest, factory_boy, coverage, pre-commit, GitHub Actions

**DoD spécifique :** `docker compose up` lance tout, Swagger est accessible, la CI est verte.

---

## Phase 2 — Comptes & authentification

- [ ] Inscription, connexion (JWT), refresh, logout (blacklist)
- [ ] Vérification de l'email par **code OTP à 6 chiffres envoyé par email** (décidé : `USER_STORIES.md` § 8)
- [ ] Réinitialisation du mot de passe
- [ ] Profil `/me/` (nom, téléphone)
- [ ] Carnet d'adresses (ville, quartier, repère, adresse par défaut)
- [ ] Throttling sur les endpoints d'authentification
- [ ] (Option) Login Google

---

## Phase 3 — Vendeurs

- [ ] Candidature vendeur : nom de boutique, ville, téléphone Mobile Money pour les reversements
- [ ] Upload des pièces KYC (privées, jamais publiques) : identité + selfie + un justificatif d'activité (RCCM, patente ou NIU)
- [ ] Acceptation des CGU vendeur (version + date), dont la clause de pénalités
- [ ] Validation par l'admin : `pending → approved / rejected → suspended`
- [ ] Profil public de la boutique (logo, description, ville)
- [ ] Adresse de ramassage de la boutique (quartier, repère, téléphone) — jamais publique
- [ ] Permission `IsApprovedSeller`

---

## Phase 4 — Catalogue

- [ ] Catégories hiérarchiques (django-treebeard)
- [ ] Attributs : taille, **état (grade A/B/C)**, marque, genre, couleur
- [ ] `Listing` (pièce unique, `quantity` = 1 par défaut). Statuts : `draft → published → reserved → sold`, plus `archived` et `removed` (modération)
- [ ] Images multiples (ordre, image principale, compression et miniatures en Celery)
- [ ] Endpoints vendeur : CRUD de ses annonces (anti-BOLA)
- [ ] Endpoints publics : liste, détail, filtres (catégorie, taille, état, prix, ville), tri
- [ ] Recherche full-text PostgreSQL
- [ ] Modération par l'admin (retrait d'annonce + motif)

**Jalon v0.3** : mise en ligne pour l'onboarding des vendeurs.

---

## Phase 5 — Panier

- [ ] Panier invité (`X-Cart-Token`) et panier connecté
- [ ] Fusion au login
- [ ] Ajouter et retirer une annonce, avec vérification de sa disponibilité
- [ ] Panier multi-vendeurs (regroupement par vendeur dans la réponse)
- [ ] Gestion des annonces vendues entre-temps (marquées indisponibles)

---

## Phase 6 — Checkout & commandes

- [ ] **Villes et tarifs de livraison** (intra-ville : livreur FripShop ; inter-ville : livreur + agence de voyage), gérés dans l'admin — prérequis du checkout (US-A-12, déplacé depuis la Phase 9)
- [ ] Frais de livraison calculés par `SellerOrder` (déplacé depuis la Phase 9)
- [ ] Checkout : adresse → livraison (par vendeur) → récapitulatif → paiement
- [ ] `Order` → `SellerOrder` → `OrderItem` (snapshot)
- [ ] Création **transactionnelle** et **réservation** des annonces (`select_for_update`) avec expiration
- [ ] Machine à états de `SellerOrder` : `pending_payment → paid → accepted → shipped → delivered → completed` (+ `cancelled`, `refunded`, `disputed`)
- [ ] Numéro lisible (`CMD-2026-000152`, sous-commande `CMD-2026-000152-A`)
- [ ] Côté client : historique des commandes
- [ ] Côté vendeur : liste et acceptation de ses sous-commandes (sans les coordonnées du client)
- [ ] Facture / reçu PDF

---

## Phase 7 — Paiements

- [ ] Interface `PaymentGateway` + adaptateur `CampayGateway` (sandbox d'abord, ADR 0004 § 8.1)
- [ ] Remboursements via `withdraw` vers le payeur (pas de remboursement natif chez Campay)
- [ ] Tâche de rapprochement des `Transaction` et `Payout` en attente (pas de relance automatique d'un `withdraw`)
- [ ] `Transaction` (audit complet)
- [ ] Initiation du paiement Mobile Money + **webhooks** (signature, idempotence)
- [ ] Échecs, expirations, remboursements
- [ ] Celery Beat : annulation des commandes impayées et libération des annonces
- [ ] Paiement confirmé après l'expiration : re-réservation si les annonces sont libres, sinon remboursement automatique

---

## Phase 8 — Commissions & reversements

- [ ] Calcul de la commission par `SellerOrder` (10 % sur les articles seuls, figée au paiement)
- [ ] **Registre** des mouvements du solde vendeur (crédit en attente, disponible, reversé)
- [ ] Libération des fonds : livraison + délai sans réclamation (Celery Beat)
- [ ] `Payout` vers Mobile Money via la passerelle (webhooks idempotents)
- [ ] Endpoint vendeur : solde, historique des gains et des reversements
- [ ] Gestion des reversements dans l'admin
- [ ] **Pénalités vendeur** (`Penalty`) : déduction automatique du solde, règlement à chaque libération, paiement par Mobile Money
- [ ] Suspension automatique au-delà de 3 pénalités impayées + mise en demeure ; réactivation automatique

---

## Phase 9 — Livraison

> Les villes, les tarifs et le calcul des frais sont en **Phase 6** : le checkout en a besoin.

> **Décision du 2026-10-20 :** FripShop livre avec ses propres livreurs (ERD § 3.9, § 4.8). Les livreurs sont payés hors plateforme.

- [ ] Rôle **Livreur** (`Courier`) attribué et suspendu par l'admin ; permission `IsActiveCourier` ; espace `/api/v1/courier/…`
- [ ] `Delivery` + machine à états (`pending_assignment → assigned → picked_up → delivered`, `deposited`, `failed_attempt`, `returned_to_seller`)
- [ ] Attribution manuelle des livraisons par l'admin (livreur de la ville, jamais l'acheteur ni le vendeur)
- [ ] Endpoints livreur : mes livraisons, ramassage, remise avec **code client**, échec, dépôt en agence (bordereau)
- [ ] Code de livraison à 4 chiffres (5 essais) envoyé au client à l'acceptation
- [ ] Visibilité des coordonnées par rôle (vendeur : jamais ; livreur : pendant la livraison)
- [ ] Inter-ville : confirmation par le client ou automatique 7 jours après le dépôt en agence
- [ ] Suivi côté client et côté vendeur ; historique des courses du livreur

---

## Phase 10 — Litiges & retours *(version minimale dans le MVP)*

- [ ] Ouverture d'un litige par le client dans le délai (motif, photos)
- [ ] Gel des fonds **de la sous-commande contestée** uniquement (le reste du solde reste reversable)
- [ ] Résolution par l'admin : remboursement total, partiel ou rejet, avec **cause attribuée** (vendeur, FripShop, client) ; pénalité si faute du vendeur
- [ ] Historique des échanges

---

## Phase 11 — Back-office & statistiques

- [ ] Admin Django personnalisé : vendeurs, modération, commandes, litiges, reversements
- [ ] Endpoints de statistiques vendeur : ventes, revenus, annonces vues ou vendues
- [ ] Statistiques admin : volume de ventes, commissions, vendeurs actifs

---

## Phase 12 — Notifications

- [ ] Emails transactionnels (HTML, Celery)
- [ ] **Push mobile** (FCM) : nouvelle commande (vendeur), expédition, livraison (client)
- [ ] Enregistrement des tokens d'appareil
- [ ] (Option) SMS via un fournisseur local

---

## Phase 13 — Croissance *(post-MVP)*

- [ ] Avis sur les vendeurs (acheteurs vérifiés uniquement)
- [ ] Favoris et suivi de boutiques
- [ ] « Faire une offre » (négociation)
- [ ] Coupons et promotions
- [ ] Recommandations et annonces similaires
- [ ] Mise en avant payante d'annonces

---

## Phase 14 — Durcissement

- [ ] `check --deploy` sans avertissement
- [ ] **OWASP API Security Top 10** (BOLA, authentification, exposition excessive de données, rate limiting…)
- [ ] Revue des permissions de chaque endpoint
- [ ] Optimisation SQL (N+1, index), cache des listes publiques
- [ ] Tests du parcours d'achat complet de bout en bout (API) — `USER_STORIES.md` § 9
- [ ] Test de charge sur la liste et la recherche d'annonces

---

## Phase 15 — Déploiement & exploitation

- [ ] Hébergement (VPS ou PaaS)
- [ ] Gunicorn + Nginx + HTTPS
- [ ] Médias sur stockage S3-compatible (+ CDN si besoin)
- [ ] Backups PostgreSQL + test de restauration
- [ ] Sentry + logs structurés
- [ ] CD depuis `main`
- [ ] Domaine, emails transactionnels (SPF/DKIM)
- [ ] Publication du schéma OpenAPI pour les équipes front

---

## Pistes front *(repos séparés, hors de cette roadmap)*

| Piste | Dépend de |
|---|---|
| Web React : catalogue, panier, checkout, compte client, SEO | Contrat API (Phase 0), puis chaque phase livrée |
| Web React : espace vendeur | Phases 3, 4, 6, 8 |
| App mobile : client + vendeur | Mêmes phases, plus les push (Phase 12) |

> Le front peut avancer en parallèle grâce au schéma OpenAPI et à des mocks.
