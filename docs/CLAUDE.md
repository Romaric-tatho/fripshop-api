# CLAUDE.md — Règles d'architecture et conventions du projet

> Ce fichier est la **source de vérité technique**. Tout code produit (par un humain ou par Claude) doit le respecter.
> Il est placé à la racine du repo (lu automatiquement par Claude Code) **et** dans les connaissances du projet Claude.

---

## 1. Présentation

**FripShop** est une marketplace de friperie (fripiers professionnels vérifiés), pour le **Cameroun**, en devise **XAF**.
Le backend est une **API REST** : **Django 5.x + Django REST Framework**, avec une authentification **JWT** (simplejwt).
Ses clients sont un site web **React** et une **app mobile**, chacun dans un repo séparé.

Voir aussi : `docs/ROADMAP.md` · `docs/PROGRESS.md` · `docs/VISION.md` · `docs/USER_STORIES.md` · `docs/ERD.md` · `docs/API.md` · ADR `docs/adr/`.

---

## 2. Structure du repo

```
fripshop-api/
├── CLAUDE.md
├── README.md
├── .env.example
├── .pre-commit-config.yaml
├── pyproject.toml              # ruff, black, mypy, pytest
├── requirements/{base,dev,prod}.txt
├── Dockerfile
├── docker-compose.yml          # web, db, redis, celery, celery-beat
├── .github/workflows/ci.yml
├── docs/
│   ├── ROADMAP.md  PROGRESS.md  VISION.md  USER_STORIES.md  ERD.md  API.md
│   └── adr/
├── config/
│   ├── settings/{base,dev,test,prod}.py
│   ├── urls.py                 # /admin/, /api/v1/, /api/schema/, /api/docs/
│   ├── api_urls.py             # agrège les urls.py de chaque app sous /api/v1/
│   ├── celery.py
│   ├── wsgi.py  asgi.py
├── apps/
│   ├── core/          # TimeStampedModel, UUIDModel, City, AuditLog, pagination, exception handler, permissions communes
│   ├── accounts/      # User (login email), profil, adresses, auth JWT, OTP
│   ├── sellers/       # Seller (boutique), vérification KYC
│   ├── catalog/       # Category, Listing (pièce unique), ListingImage, attributs
│   ├── cart/          # panier (invité via cart token + connecté)
│   ├── orders/        # Order → SellerOrder → OrderItem, machine à états
│   ├── payments/      # PaymentGateway, Transaction, webhooks
│   ├── payouts/       # commissions, registre du solde vendeur, reversements
│   ├── shipping/      # tarifs, livreurs (Courier), livraisons (Delivery)
│   ├── disputes/      # litiges et retours
│   ├── notifications/ # emails, push (FCM), SMS
│   ├── dashboard/     # endpoints de statistiques vendeur et admin
│   ├── reviews/       # post-MVP
│   └── promotions/    # post-MVP
└── templates/
    └── emails/        # seuls templates du projet
```

### Structure interne d'une app

```
apps/catalog/
├── models.py         # données + règles liées aux données uniquement
├── services.py       # logique métier en ÉCRITURE
├── selectors.py      # logique métier en LECTURE
├── serializers.py    # validation d'entrée et représentation de sortie, SANS logique métier
├── views.py          # vues API minces (APIView / generics)
├── permissions.py    # permissions spécifiques (ex. IsListingOwner)
├── filters.py        # django-filter
├── urls.py           # app_name = "catalog"
├── admin.py
├── tasks.py          # Celery
├── exceptions.py     # exceptions métier
└── tests/
    ├── factories.py
    ├── test_models.py
    ├── test_services.py
    ├── test_selectors.py
    └── test_api.py   # endpoints + permissions
```

---

## 3. Règles d'architecture (non négociables)

1. **Custom User** (`accounts.User`, login par email) est créé avant la première migration. On référence toujours `settings.AUTH_USER_MODEL`. Un vendeur est un `User` qui possède un `Seller`.
2. **Les vues sont minces.** Une vue :
   - valide l'entrée avec un serializer **d'entrée** ;
   - appelle un **service** (écriture) ou un **selector** (lecture) ;
   - renvoie la réponse via un serializer **de sortie**.

   Aucune logique métier dans les vues ni dans les serializers : pas de `serializer.save()` qui contient des règles, pas de `create()` ni `update()` métier.
3. **Les services** sont des fonctions explicites, avec des arguments nommés et des types.
   ```python
   def order_create(*, user: User, cart: Cart, address: Address) -> Order: ...
   ```
4. **Argent** : `DecimalField(max_digits=12, decimal_places=2)` avec une devise explicite. **Jamais de `float`.** Dans l'API, un montant s'écrit `{"amount": "15000.00", "currency": "XAF"}`, avec le montant en chaîne.
5. **Snapshot** : `OrderItem` copie le titre, la référence, le prix unitaire et le vendeur au moment de l'achat.
6. **Pièce unique et stock** : une `Listing` a une `quantity` (1 par défaut). Toute réservation ou vente passe par `transaction.atomic()` + `select_for_update()`.
7. **Commandes** :
   - un paiement = un `Order` ;
   - un `Order` contient un `SellerOrder` par vendeur ;
   - les changements de statut se font au niveau de `SellerOrder`, via une **machine à états** (transitions autorisées uniquement, voir `docs/ERD.md` § 4).
8. **Argent des vendeurs** :
   - FripShop encaisse, puis la commission et le solde sont enregistrés dans un **registre** (écritures immuables, pas de champ « solde » modifié directement) ;
   - les reversements (`Payout`) passent par la passerelle de paiement ;
   - les fonds ne sont libérés qu'après la livraison et l'expiration du délai de réclamation ;
   - un litige ne gèle que les fonds de la sous-commande contestée.
9. **Paiements** : ils passent derrière l'interface `PaymentGateway`. Les webhooks ont une signature vérifiée et sont **idempotents**, y compris ceux des reversements.
10. **Permissions explicites** :
    - par défaut, `DEFAULT_PERMISSION_CLASSES = [IsAuthenticated]` ;
    - chaque vue déclare ses `permission_classes` ;
    - tout accès à un objet appartenant à quelqu'un (annonce, commande, adresse) vérifie le propriétaire (**anti-BOLA/IDOR**) et c'est **testé** ;
    - l'objet d'un autre utilisateur renvoie **404**, jamais 403 : on ne révèle pas son existence. Le plus simple est de filtrer le queryset par propriétaire dans le selector (`listing_get_for_seller`, `order_get_for_user`).
11. **Identifiants publics** : l'API expose des **UUID** (`public_id`) ou des slugs, jamais l'`id` auto-incrémenté.
12. **Tâches lentes** (emails, push, appels externes, traitement d'images) → Celery. On passe des **IDs** aux tâches, jamais des objets.
13. **Secrets** : uniquement dans `.env`, via django-environ.
14. Tous les modèles héritent de `core.models.TimeStampedModel` (`created_at`, `updated_at`).
15. **Exceptions métier** :
    - elles sont définies dans le `exceptions.py` de chaque app et héritent de `core.exceptions.ApplicationError` ;
    - le **gestionnaire d'erreurs global** les convertit en réponses HTTP au format standard (§ 5).
16. **Règles chiffrées** (délais, minimums, limites) : des réglages dans `settings` (valeurs de `docs/USER_STORIES.md` § 8), jamais des constantes en dur.

---

## 4. Authentification

- `djangorestframework-simplejwt` : access token de **15 min**, refresh token de **7 jours**, **rotation + blacklist**.
- Endpoints : `/api/v1/auth/register/`, `login/`, `refresh/`, `logout/` (blacklist), `verify/`, `password-reset/`, `password-reset/confirm/`.
- **Vérification et réinitialisation** par **code OTP à 6 chiffres envoyé par email** (10 min, 5 essais, code stocké haché). Un compte non vérifié peut se connecter mais pas commander.
- **Throttling** strict sur l'inscription, la connexion, l'OTP, la réinitialisation et le checkout.

---

## 5. Conventions API

- **Préfixe** : `/api/v1/`. Ressources au **pluriel**, en **kebab-case** : `/api/v1/listings/`, `/api/v1/seller-orders/`.
- **Espaces** :
  - public : `/api/v1/listings/` ;
  - client connecté : `/api/v1/me/…` (`/me/orders/`, `/me/addresses/`) ;
  - vendeur : `/api/v1/seller/…` (`/seller/listings/`, `/seller/orders/`) ;
  - livreur : `/api/v1/courier/…` (`/courier/deliveries/`).
- **Un serializer de sortie par espace** quand les droits de lecture diffèrent (ex. coordonnées du client : jamais côté vendeur, seulement pendant la livraison côté livreur). On n'expose que les champs autorisés ; on ne masque pas des champs d'un serializer partagé.
- **Contrat** : la liste des endpoints, permissions et codes d'erreur est dans `docs/API.md` ; le schéma OpenAPI généré doit lui rester conforme.
- **Changements de statut** : toujours par une **action** `POST …/<verbe>/` (`/publish/`, `/accept/`, `/pickup/`) qui appelle le service de transition ; un `status` n'est jamais modifiable par `PATCH`.
- **Idempotence** : en-tête `Idempotency-Key` (UUID) obligatoire sur les `POST` qui mettent de l'argent en mouvement (checkout, paiement, reversement, pénalité) ; même clé → même réponse pendant 24 h.
- **Pagination** : `PageNumberPagination`, 20 éléments par défaut, 100 au maximum. Format : `{count, next, previous, results}`.
- **Format d'erreur unique** :
  ```json
  {"error": {"code": "insufficient_stock", "message": "Cet article n'est plus disponible.", "details": {}}}
  ```
- **Dates** : ISO 8601, stockées en UTC (`USE_TZ = True`). Fuseau d'affichage : `Africa/Douala`.
- **Codes HTTP** : 200 lecture, 201 création, 204 suppression, 400 validation, 401 non authentifié, 403 interdit (rôle ou statut insuffisant, ex. vendeur non approuvé), 404 introuvable **ou appartenant à un autre utilisateur**, 409 conflit métier (article déjà vendu, transition d'état interdite), 423 verrouillé (saisie du code de livraison bloquée), 429 throttling, 502 agrégateur de paiement injoignable.
- **Documentation** : chaque vue est décrite avec `@extend_schema` (drf-spectacular). `manage.py spectacular --validate` ne doit produire **aucun warning**.
- **Messages** affichés à l'utilisateur : en **français**, via `gettext_lazy`.

---

## 6. Conventions de code

- **Python 3.12**, formatage **black**, lint **ruff**, typage **mypy** (django-stubs + djangorestframework-stubs).
- Annotations de type sur toutes les fonctions publiques.
- **Nommage** :
  - modèles : `PascalCase` au singulier (`Listing`, `SellerOrder`) ;
  - services : `<objet>_<action>` (`listing_publish`, `cart_add_item`) ;
  - selectors : `<objet>_<requête>` (`listing_list_published`, `order_get_for_user`) ;
  - vues : `<Objet><Action>Api` (`ListingListApi`, `ListingCreateApi`) ;
  - serializers : `<Objet><Action>InputSerializer` / `<Objet><Action>OutputSerializer` ;
  - noms d'URL : `catalog:listing-list`, `catalog:listing-detail`.
- Le code et les noms techniques sont en **anglais** ; les textes destinés à l'utilisateur sont en **français**.
- Docstrings courtes sur les services, qui expliquent la règle métier.
- Pas de requête SQL dans une boucle : `select_related` / `prefetch_related` dans les selectors.

---

## 7. Tests

- `pytest` + `pytest-django` + `factory_boy`, avec `APIClient` pour les endpoints.
- Chaque service et selector est testé (cas nominal et cas d'erreur).
- Chaque endpoint est testé : succès, validation, **non authentifié (401)** et **accès à l'objet d'un autre utilisateur (404)**.
- Chaque machine à états est testée : toutes les transitions autorisées, et au moins une transition interdite par état (409).
- Couverture minimale de **80 %** (`pytest --cov --cov-fail-under=80`).
- Les appels externes (paiement, reversement, email, push, SMS) sont **mockés**.

---

## 8. Git

- **Branches** : `main` (prod), `develop` (intégration), `feature/<phase>-<sujet>` (ex. `feature/p4-listings`).
- **Conventional Commits** : `feat(catalog): …`, `fix(cart): …`, `test`, `refactor`, `docs`, `chore`, `ci`.
- Une PR = une feature. La CI doit être verte avant le merge.

---

## 9. Commandes utiles

```bash
docker compose up -d
docker compose exec web python manage.py makemigrations
docker compose exec web python manage.py migrate
docker compose exec web pytest --cov
docker compose exec web python manage.py spectacular --validate --file schema.yml
pre-commit run --all-files
docker compose logs -f celery
```
