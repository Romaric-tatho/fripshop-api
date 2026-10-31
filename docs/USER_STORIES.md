# 📋 USER STORIES — FripShop API

> Livrable **P0-2**. Ce document sert de base à l'ERD (P0-3), à la liste des endpoints (`API.md`) et aux tests d'acceptation.
> Chaque story renvoie à une **phase** de `ROADMAP.md`. Les valeurs chiffrées marquées **(H)** sont des hypothèses à valider (§ 8). Elles deviendront des **réglages** (`settings`), jamais des constantes en dur.

---

## 1. Conventions

**Format :** *En tant que* `<rôle>`, *je veux* `<action>` *afin de* `<bénéfice>`.

**Identifiants :** `US-<rôle>-<nn>`, avec `V` visiteur, `C` client, `S` vendeur, `A` admin et `Y` système.

**MoSCoW :**

| Priorité | Sens pour FripShop |
|---|---|
| **M** — Must | Indispensable à la **v1.0 (MVP)**. Sans elle, l'achat payé, livré et reversé ne fonctionne pas, ou la plateforme n'est pas sûre. |
| **S** — Should | Prévue dans la v1.0, mais le lancement reste possible sans (contournement manuel ou via l'admin). |
| **C** — Could | v1.x (Phase 13) ou plus tard, si le temps le permet. |
| **W** — Won't | Explicitement **hors périmètre** de ce cycle (§ 7). |

**Critères transverses** (valables pour toutes les stories, non répétés) :
- un accès non authentifié à une ressource privée renvoie **401** ;
- l'accès à l'objet d'un autre utilisateur renvoie **404** (on ne révèle pas son existence) ;
- les erreurs suivent le format unique de `CLAUDE.md` § 5 ;
- les montants sont en XAF, au format `{"amount": "…", "currency": "XAF"}` ;
- les messages affichés sont en français.

---

## 2. Rôles

| Rôle | Description | Accès API |
|---|---|---|
| **Visiteur** | Personne non connectée. Parcourt le catalogue, peut remplir un panier invité. | Endpoints publics, panier via `X-Cart-Token` |
| **Client** | Utilisateur connecté avec un email vérifié. Achète, suit ses commandes, ouvre des litiges. | `/api/v1/me/…` |
| **Vendeur** | Client qui possède un `Seller`. Statuts : `pending`, `approved`, `rejected`, `suspended`. Seul un vendeur `approved` publie et vend. | `/api/v1/seller/…` (`IsApprovedSeller`, sauf la candidature) |
| **Admin** | Équipe FripShop (`is_staff`). Valide les vendeurs, modère, arbitre les litiges, suit l'argent. | Django Admin (+ endpoints de stats) |
| **Système** | Tâches automatiques (Celery Beat, webhooks). Pseudo-rôle utile pour exprimer les règles temporelles. | — |

> Un même utilisateur peut être client **et** vendeur. Un vendeur ne peut pas acheter ses propres annonces.

---

## 3. Visiteur

### Épopée V-A — Découvrir le catalogue *(Phases 3–4)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-V-01 | En tant que visiteur, je veux parcourir les annonces par catégorie afin de trouver ce qui m'intéresse. | M | 4 | Catégories hiérarchiques ; une catégorie parente inclut les annonces de ses sous-catégories ; seules les annonces `published` de vendeurs `approved` sont listées ; pagination 20/100. |
| US-V-02 | En tant que visiteur, je veux filtrer par taille, état (A/B/C), prix min/max, ville, genre, marque et couleur afin d'affiner ma recherche. | M | 4 | Filtres combinables ; la ville est celle du vendeur ; un filtre invalide renvoie 400. |
| US-V-03 | En tant que visiteur, je veux trier par date de publication ou par prix afin de voir d'abord les nouveautés ou les meilleures affaires. | M | 4 | Tri par défaut : plus récentes d'abord. |
| US-V-04 | En tant que visiteur, je veux rechercher par mots-clés afin de trouver une pièce précise (« veste en jean Levi's »). | M | 4 | Full-text PostgreSQL sur le titre, la description, la marque et la catégorie ; insensible aux accents ; combinable avec les filtres. |
| US-V-05 | En tant que visiteur, je veux voir le détail d'une annonce (photos, prix, état, taille, mensurations, description, boutique, ville) afin de décider d'acheter. | M | 4 | Accès par slug ou UUID ; une annonce `sold` reste consultable avec la mention « Vendu » mais n'est plus achetable ; une annonce `draft`, `archived` ou `removed` renvoie 404. |
| US-V-06 | En tant que visiteur, je veux voir la page publique d'une boutique et ses annonces afin de découvrir un fripier. | M | 3–4 | Nom, logo, description, ville, date d'inscription ; aucune donnée KYC ni numéro de reversement exposé ; boutique `suspended` → 404. |
| US-V-07 | En tant que visiteur, je veux voir les annonces similaires afin de continuer à explorer. | C | 13 | Même catégorie, taille et fourchette de prix. |

### Épopée V-B — Panier invité et inscription *(Phases 2, 5)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-V-10 | En tant que visiteur, je veux ajouter des annonces à un panier sans créer de compte afin de ne pas être freiné. | M | 5 | Le premier ajout renvoie un `cart_token` à renvoyer dans `X-Cart-Token` ; le panier invité expire après **30 jours (H)** d'inactivité. |
| US-V-11 | En tant que visiteur, je veux créer un compte avec mon email, mon nom, mon téléphone et un mot de passe afin de pouvoir commander. | M | 2 | Email unique (insensible à la casse) ; validateurs de mot de passe Django ; un code de vérification est envoyé ; throttling strict. |
| US-V-12 | En tant que visiteur, je veux me connecter avec Google afin de gagner du temps. | C | 2 | Option, après le MVP. |

---

## 4. Client

### Épopée C-A — Compte *(Phase 2)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-C-01 | En tant que client, je veux vérifier mon email avec un code reçu par email afin d'activer mon compte. | M | 2 | **Code OTP à 6 chiffres (H)** valable **10 min (H)**, **5 essais (H)** ; renvoi limité (throttling) ; un compte non vérifié peut se connecter mais **pas commander**. |
| US-C-02 | En tant que client, je veux me connecter et me déconnecter afin d'accéder à mon espace en sécurité. | M | 2 | JWT access 15 min / refresh 7 j avec rotation ; le logout blackliste le refresh ; message d'erreur identique que l'email existe ou non. |
| US-C-03 | En tant que client, je veux réinitialiser mon mot de passe oublié afin de récupérer mon compte. | M | 2 | Par code OTP ; réponse identique que l'email existe ou non ; tous les refresh tokens sont révoqués après le changement. |
| US-C-04 | En tant que client, je veux consulter et modifier mon profil (nom, téléphone) afin de garder mes informations à jour. | M | 2 | `/me/` ; l'email n'est pas modifiable par cet endpoint. |
| US-C-05 | En tant que client, je veux gérer un carnet d'adresses (ville, quartier, repère, téléphone de contact) avec une adresse par défaut afin d'être livré facilement. | M | 2 | CRUD sur **mes** adresses uniquement ; une seule adresse par défaut ; une adresse utilisée dans une commande reste intacte (snapshot dans la commande). |
| US-C-06 | En tant que client, je veux changer mon mot de passe une fois connecté afin de sécuriser mon compte. | S | 2 | Demande l'ancien mot de passe ; révoque les autres sessions. |
| US-C-07 | En tant que client, je veux supprimer mon compte afin d'exercer mon droit sur mes données. | S | 2 | Anonymisation des données personnelles ; commandes et écritures comptables conservées ; refusé si une commande ou un litige est en cours. |

### Épopée C-B — Panier *(Phase 5)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-C-10 | En tant que client, je veux ajouter une annonce à mon panier afin de l'acheter plus tard. | M | 5 | Disponibilité vérifiée : une annonce `reserved` ou `sold` renvoie **409** ; ajout idempotent (pas de doublon) ; impossible d'ajouter sa propre annonce. |
| US-C-11 | En tant que client, je veux retirer une annonce de mon panier. | M | 5 | 204 ; retirer un élément absent ne provoque pas d'erreur. |
| US-C-12 | En tant que client, je veux voir mon panier groupé par vendeur avec les sous-totaux afin de comprendre ce que je paierai. | M | 5 | Groupes par vendeur ; sous-total par vendeur et total général ; les frais de livraison apparaissent au checkout. |
| US-C-13 | En tant que client, je veux que mon panier invité soit fusionné à la connexion afin de ne rien perdre. | M | 5 | Union sans doublon ; le panier invité est ensuite supprimé. |
| US-C-14 | En tant que client, je veux être averti qu'une annonce de mon panier a été vendue entre-temps afin de ne pas être surpris au paiement. | M | 5 | L'élément est marqué `unavailable` dans la réponse et exclu du checkout. |

### Épopée C-C — Checkout et paiement *(Phases 6, 7, 9)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-C-20 | En tant que client, je veux choisir une adresse de livraison afin d'indiquer où être livré. | M | 6 | Seulement mes adresses ; snapshot de l'adresse dans la commande. |
| US-C-21 | En tant que client, je veux voir les frais et le mode de livraison de chaque vendeur afin de connaître le coût réel. | M | 9 | Calcul par `SellerOrder` selon la zone (intra-ville / inter-ville) ; une zone non desservie est signalée et bloque le checkout pour ce vendeur. |
| US-C-22 | En tant que client, je veux un récapitulatif (articles, livraison par vendeur, total) avant de payer. | M | 6 | Les montants sont recalculés côté serveur ; aucun prix envoyé par le client n'est pris en compte. |
| US-C-23 | En tant que client, je veux payer par MTN MoMo ou Orange Money afin d'utiliser mon moyen de paiement habituel. | M | 7 | Un `Order` est créé et ses annonces passent en `reserved` (transaction + `select_for_update`) ; le numéro à débiter est saisi (prérempli avec mon téléphone) ; je reçois la demande de validation sur mon téléphone ; le statut de paiement est consultable (polling) ; si une annonce n'est plus disponible → 409 et aucune commande n'est créée. |
| US-C-24 | En tant que client, je veux une confirmation avec un numéro de commande lisible afin de pouvoir la suivre. | M | 6–7 | `CMD-2026-000152` ; sous-commandes `CMD-2026-000152-A`, `-B`… ; email de confirmation. |
| US-C-25 | En tant que client, je veux pouvoir réessayer un paiement échoué tant que la réservation est valide afin de ne pas perdre mes articles. | S | 7 | Nouvelle `Transaction` sur le même `Order` ; impossible après expiration. |
| US-C-26 | En tant que client, je veux payer à la livraison dans les zones couvertes. | C | 7 | Dépend de la décision « qui encaisse ? » (§ 8). |

### Épopée C-D — Suivi des commandes *(Phases 6, 9)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-C-30 | En tant que client, je veux voir l'historique de mes commandes. | M | 6 | `/me/orders/` ; mes commandes uniquement ; tri antichronologique. |
| US-C-31 | En tant que client, je veux voir le détail d'une commande avec le statut de chaque sous-commande. | M | 6 | Articles (snapshot), vendeur, montants, statut et historique des statuts par `SellerOrder`. |
| US-C-32 | En tant que client, je veux voir les informations de suivi d'une expédition (transporteur, numéro de colis ou de bordereau). | M | 9 | Visible dès que la sous-commande est `shipped`. |
| US-C-33 | En tant que client, je veux confirmer la réception d'une sous-commande afin que le vendeur soit payé. | M | 9 | Possible uniquement en `shipped` ; passe en `delivered` et démarre le délai de réclamation. |
| US-C-34 | En tant que client, je veux annuler une sous-commande tant que le vendeur ne l'a pas acceptée afin de changer d'avis. | S | 6 | Seulement en `paid` ; remboursement automatique ; l'annonce redevient `published`. |
| US-C-35 | En tant que client, je veux télécharger un reçu PDF. | S | 6 | Généré à la demande ou en Celery ; mes commandes uniquement. |

### Épopée C-E — Litiges *(Phase 10)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-C-40 | En tant que client, je veux ouvrir un litige sur une sous-commande (non reçue, non conforme, état différent de l'annonce) avec un motif et des photos afin d'obtenir réparation. | M | 10 | Possible en `shipped` ou pendant le délai de réclamation après `delivered` ; un seul litige ouvert par sous-commande ; la sous-commande passe en `disputed` ; les fonds du vendeur sont gelés. |
| US-C-41 | En tant que client, je veux échanger des messages dans le litige et suivre sa résolution. | S | 10 | Messages visibles par le client, le vendeur concerné et l'admin uniquement. |
| US-C-42 | En tant que client, je veux être remboursé sur mon compte Mobile Money quand un litige est tranché en ma faveur. | M | 10 | Remboursement total ou partiel via la passerelle ; notification. |

### Épopée C-F — Notifications *(Phase 12)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-C-50 | En tant que client, je veux recevoir un email à chaque étape (paiement confirmé, expédition, livraison, litige) afin d'être informé. | M | 12 | Emails HTML en Celery ; aucun email ne bloque une requête API. |
| US-C-51 | En tant que client, je veux recevoir des notifications push sur l'app mobile. | S | 12 | Enregistrement et suppression du token d'appareil ; FCM. |
| US-C-52 | En tant que client, je veux recevoir les notifications importantes par SMS. | C | 12 | Fournisseur local. |

### Épopée C-G — Croissance *(Phase 13)*

| ID | Story | Prio | Phase |
|---|---|---|---|
| US-C-60 | En tant que client, je veux mettre des annonces en favoris afin de les retrouver. | C | 13 |
| US-C-61 | En tant que client, je veux suivre une boutique afin d'être averti de ses nouveautés. | C | 13 |
| US-C-62 | En tant que client ayant acheté, je veux noter et commenter un vendeur afin d'aider les autres acheteurs. | C | 13 |
| US-C-63 | En tant que client, je veux faire une offre de prix au vendeur. | C | 13 |
| US-C-64 | En tant que client, je veux appliquer un code promo. | C | 13 |

---

## 5. Vendeur

### Épopée S-A — Candidature et boutique *(Phase 3)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-S-01 | En tant que client, je veux candidater comme vendeur (nom de boutique, ville, téléphone, numéro Mobile Money et opérateur pour les reversements) afin de vendre sur FripShop. | M | 3 | Email vérifié requis ; une seule boutique par utilisateur ; nom de boutique unique ; statut initial `pending`. |
| US-S-02 | En tant que candidat, je veux déposer mes pièces KYC afin de prouver mon identité et mon activité. | M | 3 | Fichiers **privés** (jamais d'URL publique ; accès admin par URL signée à durée courte) ; formats image/PDF ; taille max **5 Mo (H)**. Pièces exigées : à décider (§ 8). |
| US-S-03 | En tant que candidat, je veux connaître le statut de ma candidature et le motif d'un refus afin de la corriger et la soumettre à nouveau. | M | 3 | `rejected` → modifiable et resoumissible, repasse en `pending` ; notification email à chaque décision. |
| US-S-04 | En tant que vendeur, je veux personnaliser ma boutique (logo, description) afin d'attirer les acheteurs. | M | 3 | Logo compressé en Celery ; slug de boutique stable. |
| US-S-05 | En tant que vendeur, je veux modifier mon numéro de reversement afin de recevoir mes gains sur le bon compte. | M | 3 | Confirmation par mot de passe **et** code OTP ; email d'alerte ; reversements **gelés 48 h (H)** après le changement (protection contre le vol de compte). |
| US-S-06 | En tant que vendeur suspendu, je veux connaître la raison de ma suspension. | S | 3 | Motif visible ; mes annonces sont masquées ; je garde l'accès à mes commandes en cours et à mon solde. |

### Épopée S-B — Annonces *(Phase 4)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-S-10 | En tant que vendeur, je veux créer une annonce en brouillon (titre, catégorie, taille, état A/B/C, marque, genre, couleur, mensurations, description, prix) afin de la préparer. | M | 4 | Statut `draft` ; prix en XAF ≥ **500 (H)** ; `quantity` = 1 par défaut. |
| US-S-11 | En tant que vendeur, je veux ajouter plusieurs photos, les ordonner et choisir la photo principale. | M | 4 | **1 à 8 photos (H)** ; compression et miniatures en Celery ; la première photo est principale par défaut. |
| US-S-12 | En tant que vendeur, je veux publier une annonce afin qu'elle apparaisse dans le catalogue. | M | 4 | Seulement si le vendeur est `approved` ; validation : au moins 1 photo et champs obligatoires remplis ; `draft` → `published`. |
| US-S-13 | En tant que vendeur, je veux modifier une annonce. | M | 4 | Mes annonces uniquement (anti-BOLA, testé) ; interdit en `reserved` ou `sold` (409) ; une modification du prix ne touche pas les paniers déjà payés (snapshot). |
| US-S-14 | En tant que vendeur, je veux archiver une annonce et la republier. | M | 4 | `published` ↔ `archived` ; impossible en `reserved`. |
| US-S-15 | En tant que vendeur, je veux lister mes annonces avec un filtre par statut. | M | 4 | `/seller/listings/` ; mes annonces uniquement. |
| US-S-16 | En tant que vendeur, je veux connaître le motif du retrait d'une annonce par la modération. | M | 4 | Statut `removed` + motif ; non republiable par le vendeur. |
| US-S-17 | En tant que vendeur, je veux supprimer un brouillon. | S | 4 | Seulement en `draft` ; les autres statuts s'archivent. |
| US-S-18 | En tant que vendeur, je veux dupliquer une annonce afin de gagner du temps sur des pièces similaires. | C | 4 | Copie en `draft`, sans les photos. |
| US-S-19 | En tant que vendeur, je veux vendre un lot de pièces identiques (quantité > 1). | C | 4 | Le modèle le permet déjà (`quantity`). |

### Épopée S-C — Commandes *(Phases 6, 9)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-S-20 | En tant que vendeur, je veux être notifié d'une nouvelle commande payée afin de la traiter vite. | M | 12 | Email au MVP (push en Should) ; uniquement après confirmation du paiement. |
| US-S-21 | En tant que vendeur, je veux lister mes sous-commandes par statut et voir leur détail (articles, adresse et téléphone de livraison). | M | 6 | `/seller/orders/` ; **mes** sous-commandes uniquement ; je ne vois que ma partie de la commande, jamais celle des autres vendeurs ; l'adresse est visible seulement après le paiement. |
| US-S-22 | En tant que vendeur, je veux accepter ou refuser une sous-commande payée. | M | 6 | Délai de **24 h (H)** ; refus = motif obligatoire, remboursement du client, annonce remise en `published` ou `archived` selon le motif. |
| US-S-23 | En tant que vendeur, je veux marquer une sous-commande comme expédiée avec le transporteur et le numéro de suivi. | M | 9 | `accepted` → `shipped` ; le client est notifié. |
| US-S-24 | En tant que vendeur, je veux télécharger un bon de préparation ou d'expédition. | C | 9 | PDF. |

### Épopée S-D — Gains et reversements *(Phase 8)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-S-30 | En tant que vendeur, je veux voir mon solde : en attente, disponible, reversé. | M | 8 | Calculé à partir du **registre** (aucun champ solde modifiable) ; montants en XAF. |
| US-S-31 | En tant que vendeur, je veux voir le détail de mes gains par sous-commande (prix, frais de livraison, commission, net). | M | 8 | Commission figée au moment de la vente. |
| US-S-32 | En tant que vendeur, je veux demander le reversement de mon solde disponible sur mon compte Mobile Money. | M | 8 | Montant ≤ disponible ; minimum de **5 000 XAF (H)** ; une seule demande en cours à la fois ; refusé si les reversements sont gelés (litige, changement de numéro, suspension). |
| US-S-33 | En tant que vendeur, je veux voir l'historique et le statut de mes reversements. | M | 8 | Statuts : `pending`, `processing`, `paid`, `failed` ; motif en cas d'échec. |
| US-S-34 | En tant que vendeur, je veux être reversé automatiquement chaque semaine. | C | 8 | Remplace ou complète US-S-32. |

### Épopée S-E — Litiges, statistiques, notifications *(Phases 10–12)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-S-40 | En tant que vendeur, je veux être notifié d'un litige et y répondre avec des preuves (photos, messages) afin de défendre ma vente. | M | 10 | Délai de réponse de **72 h (H)** ; sans réponse, l'admin tranche sur les éléments disponibles. |
| US-S-50 | En tant que vendeur, je veux voir mes statistiques (ventes, chiffre d'affaires, annonces vendues) sur une période. | S | 11 | Mes données uniquement. |
| US-S-51 | En tant que vendeur, je veux voir le nombre de vues de mes annonces. | C | 11 | Compteur agrégé, sans données personnelles. |
| US-S-52 | En tant que vendeur, je veux recevoir des notifications push (nouvelle commande, litige). | S | 12 | FCM. |
| US-S-53 | En tant que vendeur, je veux payer pour mettre une annonce en avant. | C | 13 | — |

---

## 6. Admin et système

### Épopée A — Back-office *(Phases 3, 4, 8, 10, 11)*

| ID | Story | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-A-01 | En tant qu'admin, je veux examiner une candidature et ses pièces KYC, puis l'approuver ou la rejeter avec un motif. | M | 3 | `pending` → `approved` / `rejected` ; email au candidat ; action tracée (qui, quand). |
| US-A-02 | En tant qu'admin, je veux suspendre ou réactiver un vendeur. | M | 3 | Suspension : annonces masquées, nouvelles commandes impossibles, reversements gelés ; les commandes en cours restent traitables. |
| US-A-03 | En tant qu'admin, je veux retirer une annonce avec un motif. | M | 4 | → `removed` ; le vendeur est notifié. |
| US-A-04 | En tant qu'admin, je veux gérer l'arbre des catégories et les valeurs d'attributs (tailles, marques, couleurs). | M | 4 | Une catégorie utilisée ne peut pas être supprimée, seulement désactivée. |
| US-A-05 | En tant qu'admin, je veux consulter les commandes, sous-commandes et transactions avec leur historique. | M | 6–7 | Recherche par numéro de commande, email, téléphone. |
| US-A-06 | En tant qu'admin, je veux trancher un litige : remboursement total, partiel ou rejet. | M | 10 | Écritures dans le registre ; dégel des fonds restants ; notification des deux parties ; décision motivée et tracée. |
| US-A-07 | En tant qu'admin, je veux suivre les reversements et relancer un reversement échoué. | M | 8 | Relance idempotente (pas de double paiement). |
| US-A-08 | En tant qu'admin, je veux rembourser manuellement une commande hors litige (geste commercial, erreur). | S | 7 | Motif obligatoire ; montant ≤ montant payé. |
| US-A-09 | En tant qu'admin, je veux voir les statistiques globales (volume de ventes, commissions, vendeurs actifs). | S | 11 | — |
| US-A-10 | En tant qu'admin, je veux rapprocher nos transactions avec celles de l'agrégateur. | S | 7 | Export ou vue des écarts. |
| US-A-11 | En tant qu'admin, je veux un journal des actions sensibles (validation, suspension, remboursement, reversement). | S | 11 | Qui, quoi, quand ; non modifiable. |

### Épopée Y — Système *(Phases 4, 6–10)*

| ID | Règle automatique | Prio | Phase | Critères d'acceptation |
|---|---|---|---|---|
| US-Y-01 | Une commande non payée expire après **15 min (H)** : elle est annulée et ses annonces redeviennent `published`. | M | 7 | Celery Beat ; un paiement confirmé après l'expiration déclenche un remboursement automatique. |
| US-Y-02 | Les webhooks de paiement sont vérifiés (signature) et idempotents. | M | 7 | Un webhook rejoué ne change rien ; un webhook à la signature invalide est rejeté et journalisé. |
| US-Y-03 | Les webhooks de reversement sont vérifiés et idempotents. | M | 8 | Même règle ; un échec recrédite le solde disponible par une écriture inverse. |
| US-Y-04 | Une sous-commande `paid` non acceptée en **24 h (H)** est annulée et remboursée. | S | 6 | Le vendeur et le client sont notifiés. |
| US-Y-05 | Une sous-commande `shipped` est confirmée `delivered` automatiquement après **7 jours (H)** sans action du client. | M | 9 | Rappel au client avant la confirmation automatique (Should). |
| US-Y-06 | Les fonds d'une sous-commande passent de « en attente » à « disponible » après **48 h (H)** sans litige suivant la livraison. | M | 8 | Écriture dans le registre ; la sous-commande passe en `completed`. |
| US-Y-07 | Les photos sont compressées et leurs miniatures générées en arrière-plan. | M | 4 | L'annonce reste publiable pendant le traitement (image originale en attendant). |

---

## 7. Hors périmètre (Won't, ce cycle)

| Sujet | Raison |
|---|---|
| Vente entre particuliers (C2C) | Choix de l'ADR 0002 : fripiers pros vérifiés uniquement. |
| Messagerie libre acheteur ↔ vendeur | Risque de transactions hors plateforme et de fraude ; les échanges passent par le litige. |
| Paiement par carte bancaire | Usage faible au lancement ; Mobile Money couvre la cible. |
| Multi-devise | XAF uniquement. |
| Enchères | Hors modèle « prix fixe + offre ». |
| Livraison gérée de bout en bout par un transporteur intégré via API | À réévaluer selon la décision sur la logistique (§ 8). |

---

## 8. Règles métier : hypothèses à valider

Les valeurs ci-dessous sont des **propositions** qui débloquent l'ERD et les tests. Chacune deviendra un réglage modifiable.

| Règle | Proposition | Stories | Statut |
|---|---|---|---|
| Vérification du compte | **OTP 6 chiffres par email**, 10 min, 5 essais ; SMS en option plus tard | C-01, C-03 | Proposé — tranche le point ouvert de l'ADR 0003 |
| Réservation pendant le paiement | 15 min | C-23, Y-01 | Proposé |
| Délai d'acceptation vendeur | 24 h | S-22, Y-04 | Proposé |
| Confirmation automatique de réception | 7 jours après l'expédition | Y-05 | Proposé |
| Délai de réclamation (libération des fonds) | 48 h après la livraison | C-40, Y-06 | Proposé |
| Délai de réponse du vendeur à un litige | 72 h | S-40 | Proposé |
| Gel après changement de numéro de reversement | 48 h | S-05 | Proposé |
| Reversement minimum | 5 000 XAF | S-32 | Proposé |
| Prix minimum d'une annonce | 500 XAF | S-10 | Proposé |
| Photos par annonce | 1 à 8 | S-11 | Proposé |
| Taux de commission | **à décider** | S-31 | Ouvert |
| Responsable de la livraison | **à décider** (FripShop, vendeur ou partenaire) | C-21, S-23 | Ouvert — impacte la Phase 9 |
| Pièces KYC exigées | **à décider** (ex. CNI + registre de commerce ou patente) | S-02 | Ouvert |
| Politique de retour hors litige | **à décider** (proposition : pas de retour pour « changement d'avis », seulement via litige) | C-40 | Ouvert |
| Paiement à la livraison | **Could** tant que « qui encaisse ? » n'est pas tranché | C-26 | Ouvert |

---

## 9. Parcours critique du MVP

Le MVP est atteint quand ce parcours passe de bout en bout (test API de la Phase 14) :

1. Un fripier candidate, dépose ses pièces, est approuvé (S-01, S-02, A-01).
2. Il publie une annonce avec photos (S-10, S-11, S-12).
3. Un visiteur la trouve, l'ajoute au panier, crée son compte et vérifie son email (V-04, V-10, V-11, C-01, C-13).
4. Il paie par Mobile Money ; l'annonce est réservée puis vendue (C-20 → C-24, Y-02).
5. Le vendeur accepte et expédie ; le client confirme la réception (S-22, S-23, C-33).
6. Après le délai de réclamation, les fonds sont disponibles et le vendeur se fait reverser (Y-06, S-32, Y-03).
7. Variante : le client ouvre un litige, l'admin rembourse (C-40, A-06, C-42).

---

## 10. Récapitulatif

| Rôle | Must | Should | Could |
|---|---|---|---|
| Visiteur | 8 | 0 | 2 |
| Client | 22 | 7 | 7 |
| Vendeur | 21 | 4 | 6 |
| Admin | 7 | 4 | 0 |
| Système | 6 | 1 | 0 |
| **Total** | **64** | **16** | **15** |
