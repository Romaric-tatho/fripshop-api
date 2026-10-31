# 🔌 API — Liste des endpoints (contrat v1)

> Livrable **P0-6**. Première version du contrat entre le backend et les clients (web React, app mobile).
> Ce document est la **source** du schéma OpenAPI : à chaque phase, les endpoints livrés sont décrits avec `@extend_schema`, et `schema.yml` (généré par drf-spectacular) devient la référence exacte. En cas d'écart, on corrige l'un ou l'autre dans la même PR.
> Sources : `USER_STORIES.md` (colonne **Story**), `ERD.md` (modèles et machines à états), `CLAUDE.md` § 4 et § 5 (conventions).

---

## 1. Conventions

### 1.1 Bases

| Règle | Valeur |
|---|---|
| Préfixe | `/api/v1/` (omis dans les tableaux ci-dessous) |
| Ressources | pluriel, kebab-case |
| Identifiants | `{id}` = `public_id` (UUID) ; boutiques par `{slug}` ; jamais l'`id` entier |
| Authentification | `Authorization: Bearer <access>` (JWT, 15 min) ; refresh 7 j avec rotation |
| Panier invité | en-tête `X-Cart-Token: <uuid>` |
| Idempotence | en-tête `Idempotency-Key: <uuid>` **obligatoire** sur les `POST` marqués 🔁 (voir § 1.4) |
| Pagination | `?page=&page_size=` (20 par défaut, 100 max) → `{count, next, previous, results}` |
| Tri | `?ordering=` (valeurs listées par endpoint ; `-` = décroissant) |
| Dates | ISO 8601 en UTC (`2026-10-21T15:44:00Z`) |
| Langue | messages en français ; `Accept-Language` ignoré au MVP |
| Upload | `multipart/form-data` pour les fichiers ; JSON partout ailleurs |

### 1.2 Montants

```json
{"amount": "15000.00", "currency": "XAF"}
```
Montant en **chaîne**, toujours entier en XAF. Les montants envoyés par le client ne sont **jamais** pris en compte pour un calcul (prix, frais, totaux) : ils sont recalculés côté serveur (US-C-22). Seuls les montants qui expriment un choix (prix d'une annonce, montant d'un reversement) sont acceptés en entrée.

### 1.3 Erreurs

```json
{"error": {"code": "listing_unavailable", "message": "Cet article n'est plus disponible.", "details": {"listing_id": "8f1c…"}}}
```
Erreurs de validation (400) : `code = "validation_error"`, `details = {"champ": ["message", …]}`. Catalogue des codes métier : § 9.

### 1.4 Idempotence des `POST` sensibles 🔁

Sur mobile, une requête peut partir deux fois (réseau instable, double tap). Pour les opérations qui **créent de l'argent en mouvement**, le client génère un UUID par intention et l'envoie dans `Idempotency-Key`.
- Même clé + même corps dans les 24 h → la **réponse d'origine** est renvoyée, rien n'est recréé.
- Même clé + corps différent → **409** `idempotency_key_reused`.
- Stockage : table `core.IdempotencyRecord` (clé, utilisateur, endpoint, hash du corps, réponse), purgée après 24 h. *(Ajout à l'ERD, § 10.)*

### 1.5 Permissions

| Nom | Règle |
|---|---|
| `AllowAny` | public |
| `IsAuthenticated` | JWT valide (défaut) |
| `IsVerified` | `IsAuthenticated` + email vérifié |
| `IsApprovedSeller` | possède un `Seller` `approved` |
| `IsSellerAccount` | possède un `Seller`, quel que soit son statut (consulter sa candidature, ses commandes en cours, son solde pendant une suspension) |
| `IsActiveCourier` | possède un `Courier` `active` |
| `IsStaff` | `is_staff` |

Rappels transverses (DoD) : sans JWT → **401** ; objet d'un autre utilisateur → **404** ; rôle ou statut insuffisant → **403** ; transition interdite → **409** `invalid_transition`.

---

## 2. Système

| Méthode | Chemin | Permission | Story | Réponse / remarques |
|---|---|---|---|---|
| GET | `/health/` | AllowAny | — | 200 `{status, db, redis}` |
| GET | `/schema/` · `/docs/` | AllowAny (dev) · IsStaff (prod) | — | OpenAPI 3 · Swagger UI (servis hors `/api/v1/`, voir `CLAUDE.md` § 2) |
| GET, POST | `/webhooks/campay/` | AllowAny + **signature** | Y-02, Y-03 | 200 dans tous les cas où la signature est valide (idempotent) ; 400 si signature invalide (journalisé). GET **et** POST acceptés tant que la méthode de Campay n'est pas confirmée (ADR 0004 § 7, q. 8). Le statut est **relu par l'API** Campay avant toute transition. |

---

## 3. Authentification — `/auth/`

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| POST | `/auth/register/` | AllowAny | V-11 | `{email, password, first_name, last_name, phone}` → 201 `{user}` ; envoie l'OTP | 400, 409 `email_taken` |
| POST | `/auth/login/` | AllowAny | C-02 | `{email, password}` → 200 `{access, refresh, user}` | 401 `invalid_credentials` (même message que l'email existe ou non) |
| POST | `/auth/refresh/` | AllowAny | C-02 | `{refresh}` → 200 `{access, refresh}` (rotation) | 401 `token_invalid` |
| POST | `/auth/logout/` | IsAuthenticated | C-02 | `{refresh}` → 204 (blacklist) | — |
| POST | `/auth/verify/` | IsAuthenticated | C-01 | `{code}` → 200 `{user}` | 400 `otp_invalid`, `otp_expired` ; 429 `otp_too_many_attempts` |
| POST | `/auth/verify/resend/` | IsAuthenticated | C-01 | → 204 | 409 `already_verified` ; 429 |
| POST | `/auth/password-reset/` | AllowAny | C-03 | `{email}` → 204 (**toujours**, que l'email existe ou non) | 429 |
| POST | `/auth/password-reset/confirm/` | AllowAny | C-03 | `{email, code, new_password}` → 204 ; révoque tous les refresh | 400 `otp_invalid`, `otp_expired`, `validation_error` |

---

## 4. Public — catalogue et référentiels

| Méthode | Chemin | Permission | Story | Réponse / remarques |
|---|---|---|---|---|
| GET | `/cities/` | AllowAny | C-05, V-02 | villes actives `{slug, name, region, is_served}` (non paginé) |
| GET | `/categories/` | AllowAny | V-01 | arbre complet `{slug, name, size_system, children[]}` (non paginé, mis en cache) |
| GET | `/brands/` | AllowAny | V-02 | `?q=` ; paginé |
| GET | `/catalog/attributes/` | AllowAny | V-02, S-10 | valeurs fermées : `conditions` (A/B/C + libellés), `genders`, `colors`, `sizes` par `size_system` (non paginé, mis en cache) |
| GET | `/listings/` | AllowAny | V-01 → V-04 | annonces `published` de vendeurs `approved` ; filtres ci-dessous |
| GET | `/listings/{id}/` | AllowAny | V-05 | détail ; une annonce `sold` reste visible (`is_available: false`) ; `draft`, `archived`, `removed` → 404 |
| GET | `/shops/{slug}/` | AllowAny | V-06 | nom, logo, description, ville, date d'inscription ; jamais KYC ni numéro de reversement ; boutique `suspended` → 404 |
| GET | `/shops/{slug}/listings/` | AllowAny | V-06 | mêmes filtres et tri que `/listings/` |

**Filtres de `/listings/`** : `q` (full-text, insensible aux accents), `category` (slug, **inclut les sous-catégories**), `size`, `condition` (multi : `condition=A&condition=B`), `price_min`, `price_max`, `city` (slug, ville du vendeur), `gender`, `brand` (slug), `color` (multi). Tri `ordering` : `-published_at` (défaut), `price`, `-price`. Une valeur de filtre invalide → 400.

**Représentation liste** (allégée) : `{id, slug, title, price, condition, size, brand, primary_image {url, thumbnail_url}, shop {slug, shop_name, city}, published_at, is_available}`.

---

## 5. Panier — `/cart/`

Fonctionne **invité** (`X-Cart-Token`) ou **connecté** (JWT). Connecté, le panier est celui de l'utilisateur ; l'en-tête est ignoré, sauf pour `merge`.

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET | `/cart/` | AllowAny | C-12, C-14 | → 200 panier **groupé par vendeur** : `{token, groups[{shop, items[{listing, price, is_available}], subtotal}], total, unavailable_count}` ; pas de frais de livraison ici | 404 `cart_not_found` (jeton inconnu ou expiré) |
| POST | `/cart/items/` | AllowAny | V-10, C-10 | `{listing_id}` → 201 panier ; sans jeton ni JWT, **crée** le panier et renvoie son `token` ; ajout idempotent (déjà présent → 200) | 409 `listing_unavailable`, `cannot_buy_own_listing` |
| DELETE | `/cart/items/{listing_id}/` | AllowAny | C-11 | → 204 (même si absent) | — |
| POST | `/cart/merge/` | IsAuthenticated | C-13 | en-tête `X-Cart-Token` → 200 panier fusionné ; le panier invité passe `merged` | 404 `cart_not_found` |

**Choix :** la fusion est un **appel explicite** du client juste après le login, plutôt qu'un effet de bord de `/auth/login/`. Le service d'authentification ne dépend pas du panier, et le cas se teste isolément.

---

## 6. Client connecté — `/me/`

### 6.1 Profil et adresses *(Phase 2)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET | `/me/` | IsAuthenticated | C-04 | → `{id, email, first_name, last_name, phone, email_verified, roles{seller_status, is_courier}}` | — |
| PATCH | `/me/` | IsAuthenticated | C-04 | `{first_name?, last_name?, phone?}` (email **non modifiable**) | 400 |
| POST | `/me/password/` | IsAuthenticated | C-06 | `{current_password, new_password}` → 204 ; révoque les autres sessions | 400 `invalid_password` |
| DELETE | `/me/` | IsAuthenticated | C-07 | `{password}` → 204 ; anonymisation | 409 `account_has_open_orders` |
| GET, POST | `/me/addresses/` | IsAuthenticated | C-05 | liste · création `{label, recipient_name, phone, city, neighborhood, landmark, details?, is_default?}` | 400 |
| GET, PATCH, DELETE | `/me/addresses/{id}/` | IsAuthenticated | C-05 | une adresse utilisée dans une commande reste supprimable (snapshot) | 404 |
| POST | `/me/addresses/{id}/set-default/` | IsAuthenticated | C-05 | → 200 ; l'ancienne adresse par défaut est désactivée dans la même transaction | 404 |

### 6.2 Checkout et paiement *(Phases 6–7)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| POST | `/me/checkout/preview/` | IsVerified | C-20 → C-22 | `{address_id}` → 200 récapitulatif : par vendeur `{shop, items, subtotal, shipping{method, fee, estimated_days}, is_served}`, `items_total`, `shipping_total`, `total` ; **ne réserve rien** | 403 `email_not_verified` ; 404 adresse ; 409 `cart_empty` |
| POST 🔁 | `/me/checkout/` | IsVerified | C-23, C-24 | `{address_id, payer_phone}` → 201 `{order}` (`payment_status: pending`, `expires_at`) ; réserve les annonces, crée `Order` → `SellerOrder` → `OrderItem` et lance la demande Mobile Money | 409 `listing_unavailable` (rien n'est créé), `cannot_buy_own_listing`, `address_not_served`, `cart_empty` ; 400 `invalid_phone_operator` ; 502 `payment_initiation_failed` (commande annulée, annonces libérées) |
| GET | `/me/orders/{id}/payment/` | IsAuthenticated | C-23 | → `{payment_status, expires_at, last_transaction{status, failure_message}}` — endpoint **léger** pour le polling (toutes les 3 à 5 s côté client) | 404 |
| POST 🔁 | `/me/orders/{id}/payments/` | IsVerified | C-25 | `{payer_phone}` → 201 nouvelle tentative | 409 `reservation_expired`, `order_already_paid` |

**Choix :** le choix de l'opérateur n'est pas demandé : Campay le déduit du numéro (`ER102` → 400 `invalid_phone_operator`). Le numéro est prérempli côté client avec `me.phone`.

### 6.3 Commandes *(Phases 6, 9)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET | `/me/orders/` | IsAuthenticated | C-30 | liste antichronologique `{id, number, created_at, total, payment_status, display_status}` (`display_status` calculé à partir des sous-commandes) | — |
| GET | `/me/orders/{id}/` | IsAuthenticated | C-31, C-32, C-33 | détail : adresse (snapshot), totaux, et par sous-commande : `{id, number, shop, status, items, shipping_fee, history[], delivery{status, method, courier_first_name, agency_name, waybill_number, confirmation_code}}` — `confirmation_code` présent **uniquement** pour `method = courier` et une livraison non terminale | 404 |
| GET | `/me/orders/{id}/receipt/` | IsAuthenticated | C-35 | → PDF (`application/pdf`) | 404 ; 409 `order_not_paid` |
| POST | `/me/seller-orders/{id}/cancel/` | IsAuthenticated | C-34 | → 200 ; seulement en `paid` ; remboursement automatique | 409 `invalid_transition` |
| POST | `/me/seller-orders/{id}/confirm-delivery/` | IsAuthenticated | C-33 | → 200 ; **retrait en agence uniquement** (`Delivery` `deposited`) | 409 `invalid_transition`, `confirmation_requires_code` (livraison en ville : seul le livreur valide, avec le code) |

### 6.4 Litiges *(Phase 10)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET, POST | `/me/disputes/` | IsAuthenticated | C-40 | liste · ouverture `{seller_order_id, reason, description}` → 201 | 409 `dispute_already_open`, `dispute_window_closed`, `invalid_transition` |
| GET | `/me/disputes/{id}/` | IsAuthenticated | C-41, C-42 | détail, messages, pièces jointes, résultat | 404 |
| POST | `/me/disputes/{id}/messages/` | IsAuthenticated | C-41 | `{body}` → 201 | 409 `dispute_closed` |
| POST | `/me/disputes/{id}/attachments/` | IsAuthenticated | C-40 | multipart `{file, message_id?}` → 201 (image/PDF, 5 Mo max) | 400 `file_too_large`, `file_type_not_allowed` |
| POST | `/me/disputes/{id}/withdraw/` | IsAuthenticated | ERD § 4.7 | → 200 | 409 `invalid_transition` |

### 6.5 Appareils *(Phase 12)*

| Méthode | Chemin | Permission | Story | Entrée → réponse |
|---|---|---|---|---|
| POST | `/me/devices/` | IsAuthenticated | C-51, S-52 | `{token, platform}` → 201 (idempotent sur `token`) |
| DELETE | `/me/devices/{token}/` | IsAuthenticated | C-51 | → 204 (à appeler au logout côté app) |

---

## 7. Vendeur — `/seller/`

### 7.1 Candidature et boutique *(Phase 3)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| POST | `/seller/application/` | IsVerified | S-01 | `{shop_name, city, contact_phone, pickup_neighborhood, pickup_landmark, pickup_phone, payout_provider, payout_phone, payout_account_name, terms_version}` → 201 `Seller` (`status` initial : brouillon de candidature non soumis) | 409 `seller_already_exists`, `shop_name_taken` |
| GET, PATCH | `/seller/application/` | IsSellerAccount | S-03 | lecture du statut et du motif de refus · modification si `rejected` ou non soumise | 409 `application_not_editable` |
| POST | `/seller/application/documents/` | IsSellerAccount | S-02 | multipart `{doc_type, file}` → 201 (privé) | 400 `file_too_large`, `file_type_not_allowed` |
| DELETE | `/seller/application/documents/{id}/` | IsSellerAccount | S-02 | → 204 tant que la candidature n'est pas soumise | 409 |
| POST | `/seller/application/submit/` | IsSellerAccount | S-01, S-03 | → 200 ; `→ pending` | 409 `kyc_incomplete` (`details.missing`), `invalid_transition` |
| GET, PATCH | `/seller/shop/` | IsSellerAccount | S-04, S-06 | `{description, contact_phone, pickup_*}` ; affiche `status`, `suspension_reason_code`, `suspension_reason` | — |
| PUT | `/seller/shop/logo/` | IsApprovedSeller | S-04 | multipart `{file}` → 200 (compression en Celery) | 400 |
| POST | `/seller/payout-account/change/` | IsSellerAccount | S-05 | `{current_password, payout_provider, payout_phone, payout_account_name}` → 202 ; envoie un OTP | 400 `invalid_password` |
| POST | `/seller/payout-account/change/confirm/` | IsSellerAccount | S-05 | `{code}` → 200 ; applique le changement, fixe `payout_hold_until = now + 48 h`, email d'alerte | 400 `otp_invalid`, `otp_expired` |

### 7.2 Annonces *(Phase 4)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET, POST | `/seller/listings/` | IsApprovedSeller | S-10, S-15 | liste `?status=` · création en `draft` `{title, category, brand?, size?, condition, gender, color, measurements?, description, price}` | 400 `price_below_minimum`, `validation_error` |
| GET, PATCH | `/seller/listings/{id}/` | IsApprovedSeller | S-13 | modification en `draft`, `published`, `archived` ; **`status` n'est jamais modifiable par PATCH** | 409 `listing_not_editable` (`reserved`, `sold`, `removed`) |
| DELETE | `/seller/listings/{id}/` | IsApprovedSeller | S-17 | → 204, `draft` uniquement | 409 `listing_not_deletable` |
| POST | `/seller/listings/{id}/publish/` | IsApprovedSeller | S-12, S-14 | `draft` ou `archived` → `published` | 409 `listing_incomplete` (`details.missing`), `invalid_transition` |
| POST | `/seller/listings/{id}/unpublish/` | IsApprovedSeller | ERD § 4.2 | `published → draft` | 409 |
| POST | `/seller/listings/{id}/archive/` | IsApprovedSeller | S-14 | → `archived` | 409 `invalid_transition` |
| POST | `/seller/listings/{id}/duplicate/` | IsApprovedSeller | S-18 *(C)* | → 201 copie en `draft`, sans photos | — |
| POST | `/seller/listings/{id}/images/` | IsApprovedSeller | S-11 | multipart `{file}` → 201 `{id, url, thumbnail_url: null, position, is_primary}` (miniature générée en Celery) | 409 `max_images_reached` ; 400 |
| PATCH | `/seller/listings/{id}/images/{image_id}/` | IsApprovedSeller | S-11 | `{is_primary: true}` → 200 | 404 |
| DELETE | `/seller/listings/{id}/images/{image_id}/` | IsApprovedSeller | S-11 | → 204 | 409 `last_image_of_published_listing` |
| POST | `/seller/listings/{id}/images/reorder/` | IsApprovedSeller | S-11 | `{image_ids: [uuid, …]}` (liste complète, dans l'ordre) → 200 | 400 `image_set_mismatch` |

**Choix :** les transitions de statut sont des **actions** (`POST …/publish/`), pas un `PATCH {status}`. Chaque action correspond à un service avec ses gardes et ses effets (ERD § 4), et le schéma OpenAPI documente une erreur précise par action.

### 7.3 Sous-commandes *(Phases 6, 9)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET | `/seller/orders/` | IsSellerAccount | S-21 | `?status=` ; mes sous-commandes uniquement | — |
| GET | `/seller/orders/{id}/` | IsSellerAccount | S-21, S-23, S-31 | articles, **ville de destination seulement**, statut, historique, `accept_deadline_at`, `delivery{status, courier_first_name, courier_phone}`, gains `{items_subtotal, commission_rate, commission, net}` ; **jamais** les coordonnées du client | 404 |
| POST | `/seller/orders/{id}/accept/` | IsApprovedSeller | S-22 | → 200 ; crée la `Delivery` | 409 `invalid_transition`, `accept_deadline_passed` |
| POST | `/seller/orders/{id}/refuse/` | IsApprovedSeller | S-22 | `{reason: seller_item_unavailable | seller_item_damaged | seller_refused_other, note?}` → 200 ; remboursement du client | 409 `invalid_transition` |
| GET | `/seller/orders/{id}/packing-slip/` | IsSellerAccount | S-24 *(C)* | → PDF | 404 |

### 7.4 Argent : solde, reversements, pénalités *(Phase 8)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET | `/seller/balance/` | IsSellerAccount | S-30 | → `{pending, available, in_transit, paid_out, unpaid_penalties{count, total}, payouts_blocked_reason}` | — |
| GET | `/seller/ledger/` | IsSellerAccount | S-30, S-31 | écritures du registre `?account=&entry_type=&from=&to=` | — |
| GET | `/seller/payouts/` | IsSellerAccount | S-33 | historique | — |
| POST 🔁 | `/seller/payouts/` | IsSellerAccount | S-32 | `{amount}` → 201 `{payout}` (`pending`) | 409 `payout_below_minimum`, `insufficient_available_balance`, `payout_already_pending`, `payouts_on_hold` (`details.until`), `seller_suspended` |
| GET | `/seller/payouts/{id}/` | IsSellerAccount | S-33 | statut, frais (supportés par FripShop), motif d'échec | 404 |
| POST | `/seller/payouts/{id}/cancel/` | IsSellerAccount | ERD § 4.6 | → 200, `pending` uniquement | 409 `invalid_transition` |
| GET | `/seller/penalties/` | IsSellerAccount | S-35 | `?status=` ; sous-commande, motif, montant, statut | — |
| POST 🔁 | `/seller/penalties/{id}/pay/` | IsSellerAccount | S-35 | `{payer_phone}` → 201 `{transaction}` (demande Mobile Money) | 409 `penalty_already_settled` ; 400 `invalid_phone_operator` |
| GET | `/seller/penalties/{id}/payment/` | IsSellerAccount | S-35 | polling du paiement | 404 |

**Choix :** ces endpoints utilisent `IsSellerAccount` et non `IsApprovedSeller`. Un vendeur **suspendu** doit pouvoir consulter son solde et **payer ses pénalités** pour être réactivé (Y-08). La création d'un reversement reste refusée pendant une suspension (`seller_suspended`).

### 7.5 Litiges et statistiques *(Phases 10–11)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET | `/seller/disputes/` | IsSellerAccount | S-40 | litiges sur mes sous-commandes, `seller_response_due_at` | — |
| GET | `/seller/disputes/{id}/` | IsSellerAccount | S-40 | détail, messages, pièces | 404 |
| POST | `/seller/disputes/{id}/messages/` | IsSellerAccount | S-40 | `{body}` → 201 | 409 `dispute_closed` |
| POST | `/seller/disputes/{id}/attachments/` | IsSellerAccount | S-40 | multipart → 201 | 400 |
| GET | `/seller/stats/` | IsSellerAccount | S-50 *(S)*, S-51 *(C)* | `?from=&to=` → ventes, chiffre d'affaires, annonces vendues, vues | 400 `invalid_period` |

---

## 8. Livreur — `/courier/` *(Phase 9)*

| Méthode | Chemin | Permission | Story | Entrée → réponse | Erreurs |
|---|---|---|---|---|---|
| GET | `/courier/deliveries/` | IsActiveCourier | D-01 | **mes** livraisons `?status=` (défaut : non terminales), tri par `assigned_at` | — |
| GET | `/courier/deliveries/{id}/` | IsActiveCourier | D-01 | ramassage `{shop_name, pickup_*}`, livraison `{first_name, phone, city, neighborhood, landmark, details}` **tant que non terminale** (sinon seulement la ville), méthode, tentatives ; **jamais** le code ni l'email du client | 404 |
| POST | `/courier/deliveries/{id}/pickup/` | IsActiveCourier | D-02 | → 200 ; `assigned → picked_up`, sous-commande `shipped` | 409 `invalid_transition` |
| POST | `/courier/deliveries/{id}/deliver/` | IsActiveCourier | D-03 | `{code}` → 200 ; `picked_up → delivered` | 400 `delivery_code_invalid` (`details.attempts_left`) ; 423 `delivery_code_locked` ; 409 `invalid_transition` (méthode `agency`) |
| POST | `/courier/deliveries/{id}/fail/` | IsActiveCourier | D-04 | `{reason: absent | unreachable | address_not_found | refused, note?}` → 200 | 409 `invalid_transition` |
| POST | `/courier/deliveries/{id}/retry/` | IsActiveCourier | D-04 | → 200 ; `failed_attempt → picked_up` | 409 `max_failed_attempts_reached` |
| POST | `/courier/deliveries/{id}/deposit/` | IsActiveCourier | D-05 | `{agency_name, waybill_number}` → 200 ; `picked_up → deposited` | 409 `invalid_transition` (méthode `courier`) |
| POST | `/courier/deliveries/{id}/confirm-return/` | IsActiveCourier | D-07 *(S)* | → 200 ; après décision de retour par l'admin | 409 `return_not_decided` |
| GET | `/courier/history/` | IsActiveCourier | D-06 *(S)* | `?from=&to=` → courses (ramassages, livraisons, échecs, retours) et totaux | 400 `invalid_period` |

**Choix :** le code faux renvoie **400** avec le nombre d'essais restants. Après le dernier essai, la livraison est verrouillée et renvoie **423 Locked**, ce qui distingue « code faux » de « saisie bloquée, appeler l'admin ». C'est un ajout à la liste des codes HTTP de `CLAUDE.md` § 5.

---

## 9. Admin

Au MVP, le back-office est le **Django Admin** (ADR 0003, point 5) : pas d'API REST pour les opérations internes. Chaque action d'admin appelle **le même service** que l'API (jamais de `save()` direct sur un statut), ce qui garde les gardes, les effets et l'`AuditLog`.

| Opération | Story | Écran Django Admin |
|---|---|---|
| Examiner une candidature, ses pièces (URL signées), approuver / rejeter | A-01 | `Seller` : actions `approve`, `reject` (motif) |
| Suspendre / réactiver un vendeur | A-02 | `Seller` : actions |
| Retirer / restaurer une annonce | A-03 | `Listing` : actions (motif) |
| Catégories, marques, attributs | A-04 | `Category` (treebeard), `Brand` |
| Commandes, sous-commandes, transactions, recherche | A-05 | listes avec recherche par numéro, email, téléphone |
| Trancher un litige (cause, montant) | A-06, A-15 | `Dispute` : vue « résoudre » |
| Suivre et relancer un reversement | A-07 | `Payout` : action « relancer » (nouveau `Payout`) |
| Remboursement manuel | A-08 | `SellerOrder` : action « rembourser » (motif, montant) |
| Rapprochement Campay | A-10 | vue des écarts |
| Journal des actions sensibles | A-11 | `AuditLog` en lecture seule |
| Villes et tarifs de livraison | A-12 | `City`, `ShippingRate` |
| Rôle livreur | A-13 | `Courier` : créer depuis un `User`, suspendre |
| Attribuer / réattribuer une livraison | A-14 | `Delivery` : filtre « en attente d'attribution », action « attribuer » |
| Décider un retour au vendeur (cause) | D-04, D-07 | `Delivery` : action « retour au vendeur » (`return_fault`) |
| Créer / annuler une pénalité | A-15 | `Penalty` |

Seuls endpoints REST pour le staff :

| Méthode | Chemin | Permission | Story | Réponse |
|---|---|---|---|---|
| GET | `/staff/stats/` | IsStaff | A-09 *(S)* | `?from=&to=` → volume de ventes, commissions, vendeurs actifs, livraisons |

**À réévaluer** si l'attribution des livraisons sur téléphone devient pénible dans le Django Admin : un petit espace `/staff/deliveries/` pourra être ajouté sans changer les services.

---

## 10. Catalogue des codes d'erreur métier

| Code | HTTP | Où |
|---|---|---|
| `validation_error` | 400 | partout |
| `invalid_credentials`, `token_invalid` | 401 | auth |
| `email_not_verified` | 403 | checkout |
| `seller_not_approved` | 403 | `/seller/…` réservé aux vendeurs approuvés |
| `not_found` | 404 | partout (y compris objet d'un autre utilisateur) |
| `email_taken`, `shop_name_taken`, `seller_already_exists` | 409 | inscription, candidature |
| `invalid_transition` | 409 | toute action de machine à états |
| `listing_unavailable`, `cannot_buy_own_listing`, `cart_empty`, `address_not_served` | 409 | panier, checkout |
| `reservation_expired`, `order_already_paid`, `order_not_paid` | 409 | paiement, reçu |
| `listing_not_editable`, `listing_not_deletable`, `listing_incomplete`, `max_images_reached`, `last_image_of_published_listing` | 409 | annonces |
| `application_not_editable`, `kyc_incomplete` | 409 | candidature |
| `accept_deadline_passed` | 409 | vendeur |
| `payout_below_minimum`, `insufficient_available_balance`, `payout_already_pending`, `payouts_on_hold`, `seller_suspended` | 409 | reversements |
| `penalty_already_settled` | 409 | pénalités |
| `dispute_already_open`, `dispute_window_closed`, `dispute_closed` | 409 | litiges |
| `confirmation_requires_code`, `max_failed_attempts_reached`, `return_not_decided` | 409 | livraison |
| `account_has_open_orders`, `already_verified`, `idempotency_key_reused` | 409 | divers |
| `otp_invalid`, `otp_expired`, `invalid_password`, `delivery_code_invalid`, `invalid_phone_operator`, `file_too_large`, `file_type_not_allowed`, `image_set_mismatch`, `price_below_minimum`, `invalid_period` | 400 | divers |
| `delivery_code_locked` | 423 | livraison |
| `otp_too_many_attempts`, `throttled` | 429 | auth, checkout |
| `payment_initiation_failed` | 502 | checkout, pénalités (agrégateur injoignable) |

Chaque code correspond à une exception métier (`apps/<app>/exceptions.py`, héritant de `ApplicationError`) qui porte son code HTTP.

---

## 11. Throttling

| Scope | Endpoints | Limite proposée (H) |
|---|---|---|
| `auth_login` | `/auth/login/` | 5 / min par IP **et** par email |
| `auth_register` | `/auth/register/` | 5 / h par IP |
| `otp` | `/auth/verify/`, `/auth/verify/resend/`, `/auth/password-reset/…`, `/seller/payout-account/change/…` | 5 / 10 min par utilisateur ou email |
| `checkout` | `/me/checkout/`, `/me/orders/{id}/payments/`, `/seller/penalties/{id}/pay/` | 10 / h par utilisateur |
| `payment_poll` | `/me/orders/{id}/payment/`, `/seller/penalties/{id}/payment/` | 30 / min par utilisateur |
| `delivery_code` | `/courier/deliveries/{id}/deliver/` | 10 / min par livreur (en plus des 5 essais par livraison) |
| `anon` | autres endpoints publics | 120 / min par IP |
| `user` | autres endpoints authentifiés | 300 / min par utilisateur |

Les limites sont des réglages (`REST_FRAMEWORK["DEFAULT_THROTTLE_RATES"]`), stockées dans Redis.

---

## 12. Changements induits dans les autres documents (appliqués le 2026-10-21)

- **ERD :** ajout de `core.IdempotencyRecord` (`key`, `user`, `endpoint`, `request_hash`, `response_status`, `response_body`, `expires_at` ; U (`user`, `key`)) et d'un statut initial de candidature non soumise sur `Seller` (`draft`, avant `pending`).
- **`CLAUDE.md` § 5 :** ajout du code **423** (saisie du code de livraison verrouillée), de l'en-tête `Idempotency-Key` et de la règle « les statuts se changent par des actions `POST`, jamais par `PATCH` ».

---

## 13. Récapitulatif

| Espace | Endpoints | Phases |
|---|---|---|
| Système + webhooks | 3 | 1, 7 |
| Authentification | 8 | 2 |
| Public | 8 | 4 |
| Panier | 4 | 5 |
| `/me/` | 23 | 2, 6, 7, 9, 10, 12 |
| `/seller/` | 39 | 3, 4, 6, 8, 10, 11 |
| `/courier/` | 9 | 9 |
| `/staff/` | 1 | 11 |
| **Total** | **95** lignes de contrat (une ligne peut regrouper plusieurs méthodes sur le même chemin) | |
