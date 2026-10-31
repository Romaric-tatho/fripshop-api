# ADR 0002 — Type de plateforme : marketplace multi-vendeurs (fripiers pros)

- **Date :** 2026-10-19
- **Statut :** Accepté
- **Décideur :** fondateur FripShop

---

## Contexte

FripShop est un projet de **vrai business**, centré sur la friperie (vêtements, chaussures, sacs, maison) et visant **tout le Cameroun**. Le fondateur n'a **ni boutique ni stock** existant. Trois options étaient possibles : mono-vendeur, marketplace de fripiers professionnels, ou C2C entre particuliers (type Vinted).

## Décision

FripShop sera une **marketplace B2C multi-vendeurs**. Les vendeurs sont des **fripiers professionnels vérifiés**. Les particuliers ne peuvent pas vendre dans le MVP.

## Alternatives rejetées

| Option | Raison du rejet |
|---|---|
| Mono-vendeur | Le fondateur n'a pas de stock. Il faudrait financer l'achat des balles, les photos et le stockage. |
| C2C (particuliers) | La confiance, la modération et la fraude sont trop lourdes à gérer pour un MVP. Option à reconsidérer en v2. |

## Conséquences techniques

1. **App `sellers`** : modèle `Seller` (la boutique) lié à un `User`, avec un statut de vérification (`pending → approved → suspended`) et ses pièces justificatives.
2. **Pièce unique** : chaque annonce (`Listing`) a par défaut une quantité de 1. Une quantité supérieure reste possible pour des lots identiques, surtout pour la maison. Le modèle **Produit → Variantes → SKU** de la roadmap est **remplacé** ; les détails seront fixés dans l'ERD (P0-3).
3. **Commande multi-vendeurs** : le client paie une seule fois. La commande se découpe ainsi :
   - `Order` (la commande client, un seul paiement) ;
   - `SellerOrder` (une sous-commande par vendeur, avec son statut et sa livraison) ;
   - `OrderItem` (avec snapshot).

   La machine à états s'applique au niveau de `SellerOrder`.
4. **Flux d'argent** : FripShop encaisse, prélève sa **commission**, puis **reverse** au vendeur après confirmation de la livraison. Il faut donc :
   - des modèles `Commission`, un solde vendeur (registre de mouvements) et `Payout` ;
   - un agrégateur de paiement qui sait faire des **reversements (disbursements / payouts) vers Mobile Money**. C'est un critère éliminatoire pour son choix.
5. **Paiement à la livraison** : plus complexe en marketplace, car il faut savoir qui encaisse l'argent liquide et comment il remonte. Il sera limité aux zones où la logistique est maîtrisée. Décision à prendre plus tard.
6. **Rôles** : visiteur, client, **vendeur**, admin.
7. **Impact sur la roadmap** :
   - l'espace vendeur passe de « option » à **Must** ;
   - nouvelles tâches : inscription et vérification des vendeurs, modération des annonces, commissions, reversements, litiges et retours ;
   - le jalon v1.0 sera plus long qu'en mono-vendeur.
8. **`CLAUDE.md`** : ajouter `apps/sellers/` (et éventuellement `apps/payouts/`) dans la structure du repo une fois l'ERD validé.

## Risques

- La réglementation sur l'encaissement pour compte de tiers dans la zone CEMAC est **à vérifier avec un juriste**.
- Le démarrage est un problème de « poule et œuf » : les vendeurs doivent être recrutés avant le lancement public.
