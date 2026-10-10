# CLAUDE.md — MTG Lethal

Mémoire permanente du projet. À lire en entier avant toute modification.
Si une demande entre en conflit avec ce fichier, signale le conflit au lieu de trancher seul.

---

## 1. C'est quoi MTG Lethal

Une application web personnelle qui suit la collection Magic de **Florian** et l'état réel
de ses decks Commander. Elle fait trois choses :

1. **Elle sait où est chaque carte** — dans quel deck, dans quel cartable.
2. **Elle enregistre les mouvements**, pas les états. C'est la différence avec Moxfield.
3. **Elle vérifie** — décompte à 100, identité de couleur, cartes bannies, Game Changers.

Ce n'est pas un produit. C'est un outil personnel utilisé au magasin, sur un téléphone,
avec les cartes dans les mains.

---

## 2. La règle d'or : aucune donnée de carte dans le code

Toutes les erreurs de la version précédente venaient de listes recopiées à la main.
**L'application ne contient aucune liste de cartes, aucune liste de bannies, aucune liste
de Game Changers.**

| Donnée | Source | Qui l'entretient |
|---|---|---|
| Texte, coût, type, couleur, prix, image | **API Scryfall**, en direct | personne |
| Cartes bannies | `legalities.commander === "banned"` | personne |
| Game Changers | `game_changer === true` | personne |
| Collection, decks, mouvements | **Firestore** | Florian, par l'app |
| Le code | `index.html` | Claude |

Si une future modification demande de coder une liste de cartes en dur, **c'est un signal
d'alarme**. Signale-le.

### Respect de l'API Scryfall
- Au moins **100 ms entre deux appels** (la fonction `sf()` s'en charge, ne la contourne pas).
- Chaque carte résolue est mise en cache mémoire **et** dans `localStorage` (7 jours).
- Jamais de requête dans une boucle serrée sans passer par `sf()`.

---

## 3. Modèle de données

Un seul document Firestore par utilisateur : `mtg/{uid}`.

```
{
  piles:    { "DECK Ayara": { "Sol Ring": 1, "Swamp": 20, ... }, ... },
  tests:    [ { pile, entre, sort, d } ],
  arrivage: { "Nom de carte": quantité },
  journal:  [ { d, t } ],          // 400 entrées max, plus récent en premier
  partage:  [ "uid", ... ]         // comptes autorisés à LIRE ce document
}
```

**Pourquoi un seul document** : une lecture, une écriture, pas de jointure. La taille reste
très en dessous de la limite de 1 Mo. Ne pas éclater en sous-collections sans raison mesurée.

L'écriture est **débattue de 600 ms** (`sauver()`), jamais à chaque frappe.

### Noms de piles
- Un deck commence par `DECK ` — c'est ce qui déclenche le compteur à 100 (`estDeck()`).
- Les cartables et réserves portent leur nom tel quel.
- `À répartir` est le bac d'arrivage.

---

## 4. Vie privée — non négociable

**Le mot « proxy » n'apparaît nulle part** : ni dans l'interface, ni dans le code, ni dans
les noms de champs, ni dans les commentaires. Le vocabulaire est :

```
possédée · commandée · à obtenir · en test
```

« à obtenir » couvre indifféremment ce qui sera acheté et ce qui sera imprimé.

**Deux vues séparées :**
- *Vue propriétaire* — tout, statuts compris.
- *Vue partagée et export* — **quantité + nom de carte, point.** Aucun statut, aucune valeur,
  aucune colonne qui distingue une carte d'une autre. Une liste exportée ressemble à
  n'importe quelle liste de deck.

La page porte `<meta name="robots" content="noindex,nofollow">`.

Le dépôt GitHub est public (contrainte de GitHub Pages gratuit) ; **les données ne le sont
pas**. La config Firebase dans le code n'est pas un secret — ce sont les règles Firestore
qui protègent les données. Voir `firestore.rules`.

---

## 5. Le mode test — la fonction qui compte

Méthode réelle de Florian : il pose la nouvelle carte **par-dessus** l'ancienne dans le deck
pour l'essayer, et tranche plus tard. Résultat sans l'app : le deck est physiquement à plus
de 100 cartes et personne ne le sait.

L'app rend cet état visible : tant qu'un test n'est pas tranché, le deck affiche
`N en test` et son compteur reflète la carte en trop.

- **Garder** → la carte sortante quitte le deck pour le cartable.
- **Annuler** → la carte entrante quitte le deck.

Ne jamais simplifier ce flux en un simple remplacement immédiat. L'état intermédiaire *est*
la fonctionnalité.

---

## 6. Règles de construction (le domaine)

Ces règles viennent de Florian et priment sur toute suggestion :

1. **Vérifier l'identité de couleur de chaque carte.** Une carte hors couleur = erreur grave.
2. **Jamais de prix de mémoire.** Scryfall ou « prix à vérifier ». Les estimations passées
   étaient 3 à 5 fois trop basses.
3. **Vérifier les bannies** avant de proposer une carte puissante.
4. **Carte récente ou inconnue : chercher le texte exact** avant de l'expliquer.
5. **Toute décklist finale compte exactement 100 cartes.** Afficher le décompte.
6. Prix en **CAD**, réponses en **français**.
7. Pour chaque deck : bracket avec justification, Game Changers et leur nombre, combos
   infinis (cartes + tour), version « table d'amis », phrase d'annonce.

Bracket 3 = 3 Game Changers maximum. Au-delà, bracket 4. Un combo infini à deux cartes
suffit à faire un bracket 4 même avec peu de Game Changers.

---

## 7. Contraintes techniques

- **Un seul fichier** `index.html`, comme Rocky Training. Pas de build, pas de dépendances
  à installer.
- Firebase **compat** 10.7.1 par balises `<script>` (même version que Rocky).
- Même projet Firebase que Rocky : `drawing-rocky-training`. Une seule console à gérer.
- Hébergement : GitHub Pages, branche `main`, racine.
- Le CSS est dans la page. Pas de framework.
- **Priorité au téléphone** : l'app sert debout, au magasin, avec des cartes dans l'autre main.

---

## 8. Ce qui reste à faire

- Statuts par carte (`possédée / commandée / à obtenir`) — champ prévu, pas encore posé.
- Partage en lecture avec Émile et Régis (le champ `partage` existe, l'interface non).
- Détection de conflit : prévenir quand une carte est assignée à deux decks alors qu'il n'y
  en a qu'un exemplaire.
- Calcul et affichage du bracket par deck, à partir du compte de `game_changer`.
- Feuilles de jetons à imprimer (reprises de l'ancienne application).
