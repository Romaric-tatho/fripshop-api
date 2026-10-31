# 👁️ VISION — FripShop

> Document de cadrage (Phase 0). Les points marqués **[H]** sont des **hypothèses** à valider auprès de vrais fripiers et clients avant le lancement.

---

## 1. En une phrase

**FripShop est la marketplace de la friperie au Cameroun.** Les fripiers professionnels y vendent en ligne leurs pièces (vêtements, chaussures, sacs, articles de maison). Les clients de tout le pays y achètent de la seconde main de qualité **en confiance**, payée par Mobile Money ou à la livraison.

---

## 2. Le problème

### Côté acheteurs [H]
- Acheter de la friperie demande de se déplacer au marché, de fouiller longtemps et d'y passer du temps.
- Hors des grandes villes, le choix est limité.
- Les achats via WhatsApp ou Facebook sont risqués : paiement avant l'envoi, article différent des photos, vendeur injoignable.
- Il n'existe aucun moyen de chercher par taille, catégorie, état ou prix sur l'ensemble des vendeurs.

### Côté fripiers [H]
- Beaucoup vendent déjà sur les statuts WhatsApp et les groupes Facebook, mais tout est manuel : pas de catalogue, des réservations par message et le suivi « dans la tête ».
- Leur clientèle se limite à leur ville et à leurs contacts.
- Ils n'ont pas de paiement sécurisé et doivent gérer seuls les impayés et les litiges.

---

## 3. Cibles

| Rôle | Profil |
|---|---|
| **Acheteur** | 18-40 ans, urbain ou en ville secondaire, sur smartphone, paie par Mobile Money. Cherche du « bon et pas cher », des marques, des pièces uniques. |
| **Vendeur (fripier pro)** | Tient une boutique ou un étal, achète des balles, est déjà actif sur les réseaux sociaux. Veut vendre plus, plus loin, avec moins de gestion. |
| **Admin FripShop** | Valide les vendeurs, modère les annonces, gère les litiges et les reversements. |

---

## 4. Proposition de valeur et différenciation

1. **La confiance d'abord.** FripShop encaisse le paiement et ne reverse l'argent au vendeur qu'**après la réception** par le client. Tous les vendeurs sont vérifiés.
2. **Des pièces uniques avec de vraies photos.** Chaque annonce est une pièce réelle, avec un **état normalisé** (grade A / B / C), la taille et la marque.
3. **La recherche multi-vendeurs.** On filtre par catégorie, taille, état, prix et ville du vendeur, ce que WhatsApp ne permet pas.
4. **La livraison partout au Cameroun** : coursiers en ville et expédition entre villes.
5. **Un espace vendeur simple, pensé pour le mobile.** Un fripier doit pouvoir publier une pièce depuis son téléphone en moins d'une minute. *C'est un facteur clé d'adoption.*

**Alternatives actuelles du client :** marchés physiques, groupes Facebook et WhatsApp, plateformes généralistes. FripShop se distingue par la spécialisation friperie et le paiement sécurisé.

---

## 5. Modèle économique [H]

- **Commission par vente** prélevée sur le montant des articles (taux à fixer, par exemple entre 10 et 15 %).
- Plus tard : mise en avant payante d'annonces, abonnement « boutique premium ».

---

## 6. Périmètre géographique

- **Vision : tout le Cameroun.**
- **Lancement recommandé :**
  - les vendeurs sont recrutés d'abord à **Douala** (et à Yaoundé si possible) ;
  - les acheteurs peuvent commander **de partout** ;
  - la livraison se fait par coursier dans les villes des vendeurs, et par **expédition inter-urbaine** (agences de voyage ou transporteurs) vers les autres villes ;
  - hors des zones couvertes par les coursiers, le **paiement à l'avance** est obligatoire.

---

## 7. Catégories au lancement

- Vêtements : femme, homme, enfant
- Chaussures
- Sacs et accessoires
- Maison : linge, décoration, cuisine

---

## 8. Hors périmètre du MVP

- Vente entre particuliers (C2C)
- Négociation et « faire une offre » (candidat v1.x)
- Application mobile native (le site sera mobile-first ; une API viendra plus tard)
- Vente d'articles neufs

---

## 9. Indicateurs de succès [H]

À 6 mois après le lancement :
- nombre de **vendeurs actifs** (au moins une vente par mois) ;
- **volume de ventes** (GMV, en XAF) ;
- **taux de commandes livrées sans litige** ;
- délai moyen de livraison.

---

## 10. Risques principaux

| Risque | Piste |
|---|---|
| **La poule et l'œuf** : pas d'acheteurs sans vendeurs, et inversement | Recruter 15 à 20 fripiers et remplir le catalogue **avant** d'ouvrir aux acheteurs |
| Adoption par les vendeurs (habitude de WhatsApp) | Publication ultra-rapide, accompagnement terrain, aide aux photos |
| Litiges (article non conforme) | Grades d'état normalisés, photos obligatoires, paiement libéré après réception |
| Logistique hors des grandes villes | Expédition inter-urbaine, paiement à l'avance |
| **Réglementation** : encaisser de l'argent pour le compte de tiers | À vérifier auprès d'un juriste ; privilégier un agrégateur qui gère les reversements |

---

## 11. Questions ouvertes

- [ ] Taux de commission
- [ ] Paiement à la livraison : où et comment (qui encaisse l'argent liquide ?)
- [ ] Délai de libération des fonds après la livraison (par exemple 48 h sans réclamation)
- [ ] Qui assure la livraison : FripShop, le vendeur, ou un partenaire ?
- [ ] Politique de retour et de remboursement
- [ ] Documents exigés pour vérifier un vendeur (CNI, registre de commerce, NIU…)
