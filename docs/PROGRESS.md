# 📈 PROGRESS — Journal d'avancement

> Mis à jour à la fin de **chaque** conversation (Claude génère le bloc, tu le colles ici).
> Puis tu remplaces l'ancienne version dans les connaissances du projet Claude.

---

## État actuel

| Élément | Valeur |
|---|---|
| Projet | **FripShop** — marketplace de friperie, Cameroun |
| Phase en cours | Phase 1 — Fondations (Phase 0 terminée le 2026-10-21) |
| Tâche en cours | P1-1 Repo, structure, settings découpés, Docker |
| Version | — |
| Dernière mise à jour | 2026-10-21 |

---

## Décisions prises

| Date | Décision | Raison | ADR / doc |
|---|---|---|---|
| 2026-10-19 | ~~Stack : Django 5 MVT + HTMX~~ → remplacée par l'ADR 0003 | | 0001 |
| 2026-10-19 | **Backend = API REST DRF**, JWT (simplejwt), OpenAPI (drf-spectacular), clients React + app mobile, vendeurs via l'API | Deux clients (web + mobile) | 0003 |
| 2026-10-19 | Type : **marketplace B2C multi-vendeurs** (fripiers pros vérifiés), pas de C2C au MVP | Pas de stock propre ; vrai business ; C2C trop risqué pour un MVP | 0002 |
| 2026-10-19 | Catégories : vêtements, chaussures, sacs et accessoires, maison | Choix produit | — |
| 2026-10-19 | Zone : tout le Cameroun ; vendeurs recrutés d'abord à Douala (et Yaoundé) | Logistique maîtrisable au lancement | — |
| 2026-10-19 | Annonce = pièce unique (quantité 1 par défaut), au lieu de Produit → Variantes → SKU | Nature de la friperie | 0002 |
| 2026-10-19 | Vérification du compte et réinitialisation : **OTP 6 chiffres par email** (10 min, 5 essais) | Adapté au mobile ; SMS plus tard | 0003, USER_STORIES § 8 |
| 2026-10-19 | Délais (hypothèses, en `settings`) : réservation 15 min, acceptation vendeur 24 h, confirmation auto 7 j, réclamation 48 h, réponse au litige 72 h, gel après changement de numéro 48 h ; reversement min 5 000 XAF ; prix min 500 XAF ; 1 à 8 photos | Débloquer l'ERD et les tests | USER_STORIES § 8 |
| 2026-10-19 | **Paiement confirmé après expiration** : re-réservation si les annonces sont libres, sinon remboursement automatique | Fréquent avec Mobile Money ; évite des remboursements inutiles | ERD § 4.3 |
| 2026-10-19 | **Gel ciblé** pendant un litige : seuls les fonds de la sous-commande contestée restent « en attente » | Un litige ne doit pas bloquer toute l'activité d'un fripier | ERD § 4.4 |
| 2026-10-19 | Registre du solde à 4 comptes (`pending`, `available`, `in_transit`, `paid_out`), écritures immuables avec clé d'idempotence | Traçabilité, pas de double crédit | ERD § 5 |
| 2026-10-19 | Pas de statut sur `Order` ni `Shipment` : `SellerOrder.status` fait foi (+ `Order.payment_status`) | Une seule source de vérité | ERD § 3.6 |
| 2026-10-19 | `City` et `AuditLog` dans `core` | Référentiel partagé ; journal des actions sensibles | ERD § 3.1 |
| 2026-10-19 | Objet d'un autre utilisateur → **404** (pas 403) | Ne pas révéler l'existence | CLAUDE.md § 3, § 7 |
| 2026-10-19 | Villes et tarifs de livraison déplacés de la Phase 9 vers la **Phase 6** | Prérequis du checkout | ROADMAP |
| 2026-10-20 | **Agrégateur : Campay** (encaissement 2 %, reversement 1 %, J+1) ; repli : pawaPay puis CinetPay | Local, onboarding simple, coût par vente ≈ pawaPay | 0004 |
| 2026-10-20 | Webhook = notification : signature vérifiée, puis statut **relu par l'API** ; pas de relance automatique d'un `withdraw` ; remboursement = `withdraw` vers le payeur | Fiabilité, pas de double paiement | 0004 § 8.1 |
| 2026-10-20 | Les fonds restent dans le portefeuille de l'agrégateur ; seule la commission va sur le compte de FripShop | Limiter le risque réglementaire (règlement CEMAC 04/18) | 0004 § 5 |
| 2026-10-20 | **Livraison par FripShop** avec ses livreurs (rôle `Courier` attribué uniquement par l'admin) ; le vendeur ne voit jamais les coordonnées du client | Vie privée du client ; éviter les ventes hors plateforme | ERD § 3.9, § 4.8 |
| 2026-10-20 | Inter-ville : le livreur dépose en agence de voyage (bordereau), le client retire ; confirmation par le client ou auto 7 j | Couvrir tout le Cameroun sans réseau propre | ERD § 4.8 |
| 2026-10-20 | Preuve de livraison en ville : **code à 4 chiffres** donné par le client au livreur (5 essais) | Preuve solide, moins de litiges « non reçu » | ERD § 3.9 |
| 2026-10-20 | Frais de livraison au profit de FripShop ; livreurs payés **hors plateforme** ; attribution manuelle par l'admin au MVP | Rien à coder pour la paie ; simplicité | USER_STORIES § 8 |
| 2026-10-20 | **Commission 10 % sur les articles seuls** ; frais de reversement à la charge de FripShop (min. 5 000 XAF) ; reversement à la demande | Le paiement coûte ~3 % : marge d'environ 7 % | USER_STORIES § 8 |
| 2026-10-20 | Remboursement total : client à **100 %**, frais absorbés par FripShop ; livraison impossible : cause client → articles seulement ; cause vendeur → 100 % + pénalité ; cause FripShop → 100 % | Équité ; la cause est attribuée par l'admin | USER_STORIES § 8, ERD § 4.8 |
| 2026-10-20 | **Pénalité vendeur** = frais de livraison du client ; déduite du solde disponible, sinon impayée ; au-delà de 3 impayées : suspension auto + mise en demeure (15 j) ; poursuites hors plateforme | Faire payer l'erreur à celui qui l'a commise | ERD § 5.4 |
| 2026-10-20 | Pas de retour pour « changement d'avis » ; retour physique décidé par l'admin ; geste commercial payé par FripShop ; paiement à la livraison **hors v1.0** | Simplicité, coût logistique | USER_STORIES § 7, § 8 |
| 2026-10-21 | Contrat API v1 (`API.md`) : ~95 routes ; statuts changés par actions `POST` ; `Idempotency-Key` sur les `POST` d'argent ; fusion du panier par appel explicite ; admin = Django Admin appelant les services ; 423 pour le code de livraison verrouillé | Contrat stable pour le front ; pas de double paiement sur mobile | API.md |
| 2026-10-21 | Candidature vendeur en `draft` avant `pending` ; `IdempotencyRecord` dans `core` | Déposer les pièces avant de soumettre | ERD § 3.1, § 4.1 |
| 2026-10-20 | KYC : CNI ou passeport + selfie, plus un justificatif parmi RCCM, patente, NIU ; acceptation des CGU vendeur | Beaucoup de fripiers n'ont pas de RCCM | USER_STORIES S-02 |

---

## Modèles existants

*(aucun pour l'instant — le modèle cible est décrit dans `docs/ERD.md`)*

---

## Points en suspens / dette technique

- [ ] Rédiger les **CGU vendeur** (clause de pénalités) et les faire valider par le juriste
- [ ] Confirmer en exploitation les délais de `USER_STORIES.md` § 8
- [ ] Suppression de compte : anonymisation, à aligner avec la loi camerounaise sur les données personnelles
- [ ] Vérifier la réglementation sur l'encaissement pour compte de tiers (juriste) — **bloquant avant la production**
- [ ] Envoyer le questionnaire de l'ADR 0004 § 7 à Campay (questions 5, 7, 9 bloquantes avant la Phase 7) et signer le contrat
- [ ] RCCM / NIU de FripShop (exigés par Campay) : à confirmer ou lancer
- [ ] Technologie de l'app mobile (Flutter ou React Native)
- [ ] Qui développe le front web et le mobile
- [ ] Stockage du refresh token côté web (cookie HttpOnly ?)

---

## Journal

### 2026-10-21 — P0-6 Liste des endpoints (clôture de la Phase 0)
- **Fait :** contrat API v1 : conventions (montants, erreurs, idempotence, permissions), endpoints par espace (système, auth, public, panier, `/me/`, `/seller/`, `/courier/`, `/staff/`), opérations admin via Django Admin, catalogue des codes d'erreur, throttling.
- **Fichiers créés/modifiés :** `docs/API.md` (créé), `docs/ERD.md` (`IdempotencyRecord`, `Seller.draft`), `CLAUDE.md` (§ 5 : actions POST, `Idempotency-Key`, 423, 502), `docs/ROADMAP.md` (Phase 0 cochée).
- **Décisions :** voir le tableau.
- **En suspens :** CGU vendeur + juriste ; questionnaire Campay ; RCCM / NIU ; méthode HTTP des webhooks Campay (GET et POST acceptés en attendant).
- **Prochaine étape :** Phase 1 — Fondations.

### 2026-10-20 — P0-5 Règles métier (clôture)
- **Fait :** toutes les règles métier ouvertes sont tranchées : commission, frais de reversement, mode de reversement, remboursement total, livraison impossible selon la cause, pénalités vendeur et gel, retours, geste commercial, KYC, paiement à la livraison.
- **Fichiers modifiés :** `docs/USER_STORIES.md` (§ 7, § 8, S-02, S-32, S-35, A-06, A-08, A-15, D-04, Y-08 ; 75 Must, 18 Should, 14 Could), `docs/ERD.md` (`Penalty`, § 5.4, `Dispute.fault`, `Delivery.return_fault`, `Seller.suspension_reason_code`, CGU, `Transaction.penalty_charge`), `docs/ROADMAP.md` (Phases 0, 3, 7, 8, 10).
- **Décisions :** voir le tableau.
- **En suspens :** CGU vendeur + juriste ; questionnaire Campay ; RCCM / NIU de FripShop ; délais à confirmer en exploitation.
- **Prochaine étape :** P0-6 `API.md` ; Phase 1 possible en parallèle.

### 2026-10-20 — P0-5 Règles métier (partie livraison)
- **Fait :** modèle de livraison par FripShop : rôle Livreur, `Courier`, `Delivery` (remplace `Shipment`) et sa machine à états, code de livraison, visibilité des coordonnées par rôle, inter-ville via agence ; 9 nouvelles user stories (D-01 → D-07, A-13, A-14) ; registre du vendeur sans frais de livraison.
- **Fichiers modifiés :** `docs/USER_STORIES.md`, `docs/ERD.md`, `docs/ROADMAP.md` (Phases 3, 6, 9), `docs/adr/0004-agregateur-paiement.md`, `CLAUDE.md`.
- **Décisions :** voir le tableau (livraison FripShop, inter-ville en agence, code à 4 chiffres, livreurs hors plateforme, attribution manuelle).
- **En suspens :** reste de P0-5 (commission, frais de reversement, reversements à la demande, remboursement total, retours, paiement à la livraison, KYC, remboursement manuel après libération, livraison impossible).
- **Prochaine étape :** valider les autres points de P0-5, puis P0-6 `API.md`.

### 2026-10-20 — P0-4 Choix de l'agrégateur de paiement
- **Fait :** comparatif pawaPay, CinetPay, Campay, Notch Pay, Fapshi (frais, API, webhooks, KYC) ; coût par vente (~3 %) ; contrainte CEMAC/COBAC ; questionnaire ; intégration Campay détaillée.
- **Fichiers créés/modifiés :** `docs/adr/0004-agregateur-paiement.md` (créé, Accepté), `docs/ERD.md`, `docs/ROADMAP.md`.
- **Décisions :** Campay ; webhook = notification + relecture du statut ; pas de relance automatique d'un `withdraw` ; remboursement par `withdraw` ; alerte sur le solde par opérateur ; fonds laissés chez l'agrégateur.
- **En suspens :** réponses de Campay (signature des webhooks, recherche par `external_reference`, transfert entre soldes MTN et Orange) ; RCCM / NIU ; juriste.
- **Prochaine étape :** P0-5 Règles métier, puis P0-6 `API.md` ; Phase 1 possible en parallèle.

### 2026-10-19 — P0-3 ERD + machines à états
- **Fait :** modèle de données complet par app, diagramme global, 7 machines à états (Seller, Listing, Order.payment_status, SellerOrder, Transaction, Payout, Dispute), registre du solde vendeur avec exemple chiffré, délais et tâches Celery Beat, contraintes et index. Alignement avec les user stories.
- **Fichiers créés/modifiés :** `docs/ERD.md` (créé), `docs/USER_STORIES.md` (aligné), `docs/ROADMAP.md`, `CLAUDE.md`, `docs/adr/0003-architecture-api-drf.md`.
- **Décisions :** paiement tardif → re-réservation sinon remboursement ; gel ciblé pendant un litige ; registre à 4 comptes ; `City` et `AuditLog` dans `core` ; 404 sur l'objet d'un autre ; villes et tarifs en Phase 6 ; statut de l'annonce après annulation (`published`, ou `archived` si pièce indisponible ou remboursement).
- **En suspens :** commission en cas de remboursement total ; voir la liste ci-dessus.
- **Prochaine étape :** P0-4 Choix de l'agrégateur de paiement.

### 2026-10-19 — P0-2 User stories + MoSCoW
- **Fait :** rôles, 96 user stories (65 Must, 16 Should, 15 Could), hors périmètre, hypothèses de règles métier, parcours critique du MVP.
- **Fichiers créés :** `docs/USER_STORIES.md`.
- **Décisions :** OTP par email ; délais chiffrés en hypothèses (§ 8) ; pas de messagerie libre acheteur ↔ vendeur ; pas de carte bancaire au MVP.
- **Prochaine étape :** P0-3 ERD.

### 2026-10-19 — P0-1b Passage en API DRF
- **Fait :** l'architecture passe en API REST ; roadmap et conventions réécrites pour la marketplace + API.
- **Fichiers créés/modifiés :** `docs/adr/0003-architecture-api-drf.md`, `CLAUDE.md` (réécrit), `docs/ROADMAP.md` (réécrit : 16 phases, nouvelles phases Vendeurs, Commissions & reversements, Litiges).
- **Décisions :** JWT 15 min / 7 j avec rotation ; endpoints d'auth maison ; UUID publics ; format d'erreur unique ; panier invité via `X-Cart-Token` ; jalon v0.3 mis en ligne tôt pour l'onboarding des vendeurs.
- **Prochaine étape :** P0-2 User stories + MoSCoW.

### 2026-10-19 — P0-1 Vision et type de plateforme
- **Fait :** choix du type de plateforme, vision produit.
- **Fichiers créés :** `docs/VISION.md`, `docs/adr/0002-type-de-plateforme.md`.
- **Décisions :** marketplace de fripiers pros ; pièce unique ; commande découpée par vendeur (`Order` → `SellerOrder` → `OrderItem`) ; FripShop encaisse puis reverse.
- **En suspens :** voir la liste ci-dessus.
- **Prochaine étape :** P0-2 User stories (visiteur, client, vendeur, admin) et MoSCoW.

### 2026-10-19 — Initialisation
- **Fait :** roadmap, conventions (`CLAUDE.md`), instructions du projet.
- **Fichiers :** `ROADMAP.md`, `CLAUDE.md`, `PROGRESS.md`.

<!--
Modèle d'entrée :

### AAAA-MM-JJ — P<phase>-<n> <titre>
- **Fait :**
- **Fichiers créés/modifiés :**
- **Décisions :**
- **En suspens :**
- **Prochaine étape :**
-->
