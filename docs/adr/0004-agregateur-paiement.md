# ADR 0004 — Choix de l'agrégateur de paiement Mobile Money

- **Date :** 2026-10-20
- **Statut :** **Accepté** (2026-10-20) — **Campay**. Le contrat reste à signer après les réponses au questionnaire (§ 7).
- **Tâche :** P0-4 (`ROADMAP.md`, Phase 0)
- **Liens :** ADR 0002 (FripShop encaisse puis reverse), `ERD.md` § 3.7, § 3.8, § 4.5, § 4.6, § 5

---

## 1. Contexte

FripShop encaisse le paiement du client, garde l'argent jusqu'à la fin du délai de réclamation, puis reverse au fripier sa part, moins la commission (`CLAUDE.md` § 3.8). Il faut donc un prestataire qui fasse, **par API**, sur **MTN MoMo et Orange Money au Cameroun** :

1. l'**encaissement** : une demande de paiement poussée sur le téléphone du client ;
2. le **reversement** au vendeur (*payout / disbursement*) ;
3. le **remboursement** du client (annulation, litige, paiement tardif sans annonce disponible).

Ces trois opérations sont déjà modélisées dans l'ERD : `Transaction` (`charge`, `refund`), `Payout`, `WebhookEvent`, et le registre `LedgerEntry`.

## 2. Critères

| # | Critère | Poids | Pourquoi |
|---|---|---|---|
| C1 | Encaissement **et** reversement par API, MTN **et** Orange | Éliminatoire | Sans reversement par API, la Phase 8 se ferait à la main |
| C2 | Usage marketplace accepté (encaisser pour reverser à des tiers) | Éliminatoire | Modèle de l'ADR 0002 ; point réglementaire (§ 5) |
| C3 | Identifiant généré **par nous** à l'initiation (idempotence) et endpoint de **consultation du statut** | Fort | `Transaction.idempotency_key`, vérification avant d'expirer une commande (ERD § 4.3), reprise après un timeout |
| C4 | Webhooks **signés** pour l'encaissement, le reversement et le remboursement | Fort | `CLAUDE.md` § 3.9 |
| C5 | Remboursement par API | Moyen | Sinon un remboursement est un reversement vers le client, au même prix |
| C6 | Frais d'encaissement et de reversement | Fort | Impacte directement le taux de commission (point ouvert) |
| C7 | Sandbox utilisable | Moyen | Phase 7 : « sandbox d'abord » |
| C8 | Fiabilité, support, délai de règlement | Fort | Paiements bloqués = ventes perdues |
| C9 | KYC compatible avec la situation juridique de FripShop | Éliminatoire | RCCM, NIU… |

## 3. Options étudiées

Chiffres publics relevés le 2026-10-20. Ils sont **indicatifs** : les prix réels se négocient et dépendent du volume. Les taxes ne sont pas toujours incluses.

| | **pawaPay** | **CinetPay** | **Campay** | **Notch Pay** | **Fapshi** |
|---|---|---|---|---|---|
| Type | Panafricain (23 marchés) | Panafricain (zone franc) | Local (Cameroun) | Local (Cameroun) | Local (Cameroun) |
| Encaissement MTN / Orange | **1,75 % / 1,77 %** hors taxes | 2 % / 2,2 % | 2 % | ~1,5 à 2,5 % | ~3 % (standard) |
| Reversement MTN / Orange | **1,3 % / 1 %** hors taxes | 1,5 % / 1 % | 1 % | ~2 % (exemple de la doc : 100 XAF pour 5 000) | non publié |
| Reversement par API | ✅ unitaire + **en masse** | ✅ (à confirmer dans la doc API transfert) | ✅ `withdraw` | ✅ `transfers` (`cm.mtn`, `cm.orange`) | ✅ `payout` (min. 100 XAF) |
| Remboursement par API | ✅ endpoint dédié | ❓ | ❓ non documenté | ⚠️ depuis le dashboard, 5 à 7 jours ouvrés | ❓ |
| Identifiant d'idempotence fourni par nous | ✅ UUID v4 (`depositId`, `payoutId`) | ✅ `transaction_id` marchand | ✅ `external_reference` | ⚠️ pas de paramètre documenté | ✅ `external_id` |
| Consultation du statut | ✅ `FOUND` / `NOT_FOUND` explicite | ✅ | ✅ | ✅ | ✅ |
| Webhooks signés | ✅ signatures documentées | ✅ (HMAC, à vérifier) | ❓ à vérifier | ⚠️ recommandé, mais la méthode n'est pas documentée | ❓ non documenté |
| Sandbox | ✅ + numéros de test | ❓ | ✅ (montants limités à 25 XAF) | ✅ Test Mode | ✅ |
| Règlement | non publié | non publié | J+1 ouvré | J+1 à J+3 (estimation) | J+1 à J+2 (estimation) |
| KYC | Contrat **Enterprise** + échange commercial obligatoire pour le Cameroun | RCCM, secteur, chiffre d'affaires | RCCM, NIU, ACF, pièces d'identité | Vérification 1 à 2 jours | Pièce d'identité + infos sur l'activité |
| Points d'attention | Onboarding commercial plus long | Doc API transfert moins claire que celle de l'encaissement | Pas de remboursement documenté | Retours d'utilisateurs : retraits lents, transactions longtemps « pending » | SDK Python en alpha ; webhooks non documentés |

**Écartés :**
- **API directes MTN MoMo et Orange Money** : deux contrats d'entreprise, deux intégrations, deux suivis de pannes. À reconsidérer quand les volumes justifieront de supprimer la marge de l'agrégateur.
- **Flutterwave** : frais plus élevés (2 à 3,5 %) et règlement plus lent (J+2 à J+5). Il n'est utile que pour la carte bancaire, hors périmètre (`USER_STORIES.md` § 7).
- **MeSomb, PayUnit, Monetbil** : non évalués en détail faute de documentation publique sur les reversements et les remboursements. On peut les ajouter si la short-list échoue.

## 4. Coût par vente

Exemple de l'ERD § 5.2 : veste à 15 000 XAF + livraison 1 500 XAF, soit **16 500 XAF encaissés**. Reversement net au vendeur : **15 000 XAF** (commission de 10 % en hypothèse).

| Agrégateur | Encaissement | Reversement | **Total** | % du montant encaissé |
|---|---|---|---|---|
| pawaPay (MTN → MTN) | 289 | 195 | **484** | 2,9 % (hors taxes) |
| pawaPay (Orange → Orange) | 292 | 150 | **442** | 2,7 % (hors taxes) |
| Campay | 330 | 150 | **480** | 2,9 % |
| CinetPay (MTN → MTN) | 330 | 225 | **555** | 3,4 % |
| CinetPay (Orange → Orange) | 363 | 150 | **513** | 3,1 % |
| Notch Pay (estimation) | ~330 | ~300 | **~630** | ~3,8 % |

> Depuis le 2026-10-20, les frais de livraison reviennent à FripShop (livreurs internes) : le reversement net de l'exemple devient 13 500 XAF. Le coût du reversement baisse légèrement ; les ordres de grandeur ne changent pas.

**Conséquence pour la commission :** le paiement coûte **environ 3 % de chaque commande**, avant un éventuel remboursement (qui coûte un reversement de plus). Avec une commission de 10 % sur les articles (1 500 XAF), il reste environ 1 000 XAF de marge brute par vente. La commission ne doit pas descendre sous ~5 %. Un remboursement total coûte de nouveau des frais : c'est un argument pour **conserver** la commission dans ce cas (point ouvert).

## 5. Contrainte réglementaire

- Les services de paiement en CEMAC relèvent du **règlement n° 04/18/CEMAC/UMAC/COBAC** (en vigueur depuis le 1er janvier 2019). Seuls les prestataires agréés (établissements de crédit, de microfinance ou de paiement) peuvent fournir ces services. Les fonds de la clientèle doivent être **cantonnés**.
- FripShop encaisse pour le compte des vendeurs, puis leur reverse leur part. Selon la façon dont les fonds circulent, cette activité peut être assimilée à un service de paiement. Une réforme visant les agrégateurs est annoncée par la COBAC (septembre 2026), sans texte publié.
- **Choix de conception qui limite le risque :** l'argent reste dans le **portefeuille de l'agrégateur agréé**. Il ne passe jamais par un compte bancaire de FripShop avant d'être reversé. FripShop ne fait que donner des instructions (encaisser, reverser, rembourser). Seule sa commission est virée vers son compte.
- **Ce n'est pas un avis juridique.** Le point « juriste » de `PROGRESS.md` devient **bloquant avant la mise en production** (pas avant le développement).

## 6. Décision

**FripShop utilise Campay** (décision du 2026-10-20).

**Pourquoi Campay :**
- acteur local, onboarding sans contrat Enterprise ;
- reversement à 1 % sur les deux opérateurs ; coût par vente équivalent à pawaPay (§ 4) ;
- règlement J+1 ouvré, KYC clairement listé (RCCM, NIU, ACF) ;
- SDK Python maintenu (`campay` 1.1.0, janvier 2026), sandbox (`DEV`).

**Ce qu'on accepte en contrepartie** (compensé par l'intégration du § 8) :
- **pas de remboursement natif** : un remboursement est un reversement (`disburse`) vers le numéro du client, au tarif d'un reversement ;
- **signature des webhooks** décrite par des intégrateurs comme un JWT HS256 signé avec la *webhook key* de l'application, mais pas confirmée dans la documentation publique ;
- **sandbox limitée** (montants de test plafonnés à environ 25 XAF) ;
- **solde séparé par opérateur** (`mtn_balance`, `orange_balance`) : risque de manquer de solde sur un opérateur.

**Plan de repli :** pawaPay, puis CinetPay. L'interface `PaymentGateway` (§ 8) rend le changement possible sans toucher aux services métier.

## 7. Questionnaire à envoyer à Campay

Les questions 1 à 3 sont **bloquantes avant la production**. Les questions 5, 7 et 9 sont **bloquantes avant la Phase 7** (elles changent l'intégration).

1. Acceptez-vous un **marketplace** qui encaisse pour le compte de vendeurs tiers et leur reverse leur part ? Quels documents exigez-vous dans ce cas ?
2. Êtes-vous un **établissement de paiement agréé** (COBAC / ministère des Finances) ou adossé à un établissement agréé ? Les fonds marchands sont-ils **cantonnés** ?
3. Pièces KYC requises pour FripShop (RCCM, NIU, statuts…) et **délai** d'activation.
4. Frais **définitifs** pour MTN et Orange : encaissement, reversement, remboursement, taxes comprises, paliers de volume. Qui paie les frais de reversement : le marchand ou le bénéficiaire ?
5. Le solde est **séparé par opérateur** (`mtn_balance`, `orange_balance`) : peut-on **transférer** du solde MTN vers le solde Orange (et inversement), à quel coût et en combien de temps ?
6. Délai et frais de **règlement** vers notre compte bancaire (pour la commission).
7. **Remboursement** : existe-t-il un remboursement natif ? Sinon, un `disburse` vers le payeur est-il bien la méthode recommandée, et à quel tarif ?
8. **Webhooks** : schéma exact de la signature (JWT HS256 avec la webhook key ? quels claims ?), méthode HTTP (GET ou POST), politique de renvoi, IP sources. Les webhooks couvrent-ils aussi les `withdraw` ?
9. **Idempotence** : un `external_reference` déjà utilisé est-il **rejeté** ? Peut-on **retrouver une transaction par `external_reference`** quand l'appel `withdraw` a expiré sans renvoyer de `reference` Campay ?
10. **Limites** : montant minimum et maximum par transaction, plafonds journaliers par numéro.
11. **Disponibilité** : SLA, page de statut, support technique (canal, horaires).
12. **Sandbox** : numéros de test pour les cas succès, échec et timeout.

## 8. Conséquences sur le code et l'ERD

- **`PaymentGateway`** (Phase 7) est une interface indépendante du fournisseur :
  ```python
  class PaymentGateway(Protocol):
      def charge_initiate(self, *, transaction: Transaction) -> GatewayResult: ...
      def charge_status(self, *, transaction: Transaction) -> GatewayResult: ...
      def refund_initiate(self, *, transaction: Transaction) -> GatewayResult: ...
      def payout_initiate(self, *, payout: Payout) -> GatewayResult: ...
      def payout_status(self, *, payout: Payout) -> GatewayResult: ...
      def webhook_parse(self, *, headers: Mapping[str, str], body: bytes) -> WebhookEvent: ...
  ```
  - Si un fournisseur n'a pas de remboursement natif, son adaptateur implémente `refund_initiate` par un reversement vers le client.
  - Les statuts du fournisseur (`COMPLETED`, `ENQUEUED`, `IN_RECONCILIATION`…) sont **traduits dans l'adaptateur** vers nos statuts (`Transaction` § 4.5, `Payout` § 4.6).
  - Un statut inconnu n'est **jamais** traité comme un échec : il reste en attente et alerte l'admin.
- **Timeouts :** l'identifiant est enregistré **avant** l'appel à l'agrégateur. Après une erreur réseau, on consulte le statut. Un statut « introuvable » permet de marquer l'échec ; tout autre résultat laisse la transaction en attente.
- **Tâche de rapprochement** (Celery Beat, toutes les 5 min) pour les `Transaction` et `Payout` en attente depuis plus de 15 min. Un webhook peut se perdre. Cette tâche alimente aussi US-A-10.
- **Solde par opérateur :** si l'agrégateur sépare les soldes MTN et Orange, ajouter une alerte admin quand le solde d'un opérateur ne couvre pas les reversements en attente. Il faudra peut-être autoriser un vendeur à recevoir sur l'autre opérateur.
- **`Payout.fee_amount`** reçoit les frais réels renvoyés par l'agrégateur. Qui les supporte reste un point ouvert.
- **Montants entiers** : confirmé (Campay refuse les décimales), cohérent avec l'ERD § 1.
- **Choix de modèle :** `Transaction.provider` et `Payout.provider` restent des `TextChoices` ; ajouter un fournisseur ne demande qu'un nouvel adaptateur et une valeur de choix.

### 8.1 Intégration Campay (adaptateur `CampayGateway`)

| Opération FripShop | Appel Campay | Remarques |
|---|---|---|
| `charge_initiate` | `collect` asynchrone (`initCollect`) : `amount`, `currency=XAF`, `from`, `description`, `external_reference` | `external_reference` = `Transaction.idempotency_key` ; on stocke la `reference` Campay renvoyée dans `provider_reference` |
| `charge_status` | `transaction/<reference>/` | `PENDING`, `SUCCESSFUL`, `FAILED` |
| `refund_initiate` | `withdraw` (`disburse`) vers `payer_phone` | `Transaction(kind=refund)` ; coûte le tarif d'un reversement |
| `payout_initiate` | `withdraw` (`disburse`) vers `Payout.destination_phone` | API de retrait à activer dans les réglages de l'application Campay |
| `payout_status` | `transaction/<reference>/` | |
| Solde | `balance/` | `mtn_balance`, `orange_balance` séparés |

**Correspondance des statuts :**

| Campay | `Transaction` (ERD § 4.5) | `Payout` (ERD § 4.6) |
|---|---|---|
| `PENDING` | `pending` | `processing` |
| `SUCCESSFUL` | `succeeded` | `paid` |
| `FAILED` | `failed` | `failed` |
| aucune réponse après `RESERVATION_TTL` | `expired` (statut **local**, Campay n'en a pas) | — |
| statut inconnu | reste `pending` + alerte admin | reste `processing` + alerte admin |

**Règles d'intégration :**
1. **Le webhook n'est qu'une notification.** On vérifie sa signature, on l'enregistre dans `WebhookEvent` (idempotence), puis on **relit le statut** par `transaction/<reference>/` avant d'appliquer une transition. Le statut contenu dans le webhook n'est jamais utilisé directement. Cette règle nous protège même si le schéma de signature diffère de ce qui est décrit.
2. **Pas de relance automatique d'un `withdraw`.** Si l'appel expire sans réponse, le `Payout` (ou le remboursement) reste `processing` et passe dans la tâche de rapprochement. Il ne repart **jamais** automatiquement tant qu'on ne sait pas retrouver une transaction par `external_reference` (question 9). Sinon, un timeout pourrait provoquer un **double paiement**.
3. **Trésorerie Campay par opérateur** (à ne pas confondre avec le solde du vendeur dans le registre) : avant un `withdraw`, reversement ou remboursement, on vérifie le **solde Campay de FripShop** sur l'opérateur du bénéficiaire (`mtn_balance` ou `orange_balance`). S'il est insuffisant (`ER301`), l'opération reste en attente et l'admin reçoit une alerte avec le solde de chaque opérateur. Le registre du vendeur, lui, n'est jamais consulté pour un remboursement : un remboursement intervient toujours avant la libération des fonds, qui sont donc encore sur `pending`.
4. **Montants entiers** (`ER201`) : l'adaptateur envoie `int(amount)` après avoir vérifié que le `Decimal` est entier.
5. **Téléphone** au format `2376XXXXXXXX` (indicatif sans `+`, `ER101`) : conversion depuis E.164 dans l'adaptateur.
6. **Tests :** les tests unitaires mockent le client HTTP. La sandbox (montants d'environ 25 XAF) sert aux tests manuels de bout en bout, pas à la CI.
7. **SDK ou HTTP direct :** l'adaptateur appelle l'API REST avec `httpx`, en suivant la documentation Postman. Le SDK `campay` 1.1.0 sert de référence, mais sa méthode `collect` est **bloquante** jusqu'à la fin de la transaction, ce qui est incompatible avec une requête API.

## 9. Sources

- [pawaPay — grille tarifaire](https://pawapay.io/fees) · [plans](https://www.pawapay.io/plans) · [documentation](https://docs.pawapay.io/) · [reversements v2](https://docs.pawapay.io/v2/docs/payouts.md) · [mise en production](https://docs.pawapay.io/v2/docs/going_live.md)
- [VadeVise — Payment gateways in Cameroon](https://vadevise.com/en/guides/payment-gateways-cameroon/) (frais CinetPay, Campay, Fapshi ; KYC)
- [Notch Pay — Transfers](https://developer.notchpay.co/send-money/transfers) · [FAQ](https://developer.notchpay.co/faq.md)
- [Campay — documentation API (Postman)](https://documenter.getpostman.com/view/2391374/T1LV8PVA) · [SDK Python `campay`](https://pypi.org/project/campay) · [exemple d'intégration (webhook JWT HS256)](https://github.com/Backaad/sdk-mobile-money/pull/2)
- [CinetPay — documentation](https://docs.cinetpay.com/api/1.0-en/BO/virer-fond)
- [Fapshi — SDK Python](https://pypi.org/project/fapshi/)
- [Kolonell — Comparatif des agrégateurs au Cameroun 2026](https://kolonell.com/fr/blog/comparatif-agregateurs-paiement-cameroun-2026) · [Grille tarifaire 2026](https://kolonell.com/fr/blog/grille-tarifaire-agregateur-paiement-cameroun-2026)
- [NjokaLab — Intégrer MTN MoMo et Orange Money](https://www.njokalab.com/insights/how-to-integrate-mtn-momo-orange-money-cameroon-api/)
- [C'est tout droit — Règlement CEMAC sur les services de paiement](https://cesttoutdroit.com/article-droit-bancaire-et-financier/nouveau-reglement-cemac-relatif-aux-services-de-paiement/) · [NotreAfrik — COBAC, mobile money et agrégateurs](https://notreafrik.com/cobac-mobile-money-agregateurs/)
