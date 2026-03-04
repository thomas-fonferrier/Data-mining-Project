# Rapport Tâche 4 : Visualisation des données

## Objectif
Produire des visualisations permettant de comprendre la collection d’images annotées (Tâches 1–2) et les préférences des utilisateurs simulés (Tâche 3).

## Données utilisées
- `data/images_labels.json` : annotations par image (couleurs, orientation, taille, tags).
- `data/images_metadata.json` : métadonnées brutes (format de fichier, etc.).
- `data/users.json` : profils utilisateurs construits à la Tâche 3.

## Implémentation dans `project.ipynb`
Une nouvelle cellule **Tâche 4 : Visualisation des données** a été ajoutée.

### 1. Statistiques de la collection d’images
Trois graphiques dans une même figure (3 subplots) :
- **Barres – Nombre d’images par orientation**
  - Source : `labels_df["orientation"].value_counts()`.
  - But : voir la proportion de photos en paysage, portrait, carré, etc.
- **Barres – Nombre d’images par catégorie de taille**
  - Source : `labels_df["size_category"].value_counts()`.
  - But : comprendre la distribution entre `vignette`, `moyenne`, `grande`.
- **Camembert – Distribution des formats d’images**
  - Source : `metadata_df["file_format"].value_counts()`.
  - But : mesurer la part des formats (`JPG`, `PNG`, etc.) dans le dataset.

### 2. Analyse des couleurs
Deux visualisations :
- **Palette de couleurs prédominantes**
  - Récupération d’un échantillon de couleurs (RGB) dans `predominant_colors`.
  - Affichage sous forme de bande de rectangles colorés, pour visualiser la diversité générale des teintes.
- **Histogramme des fréquences de noms de couleurs**
  - Aplatissement de toutes les listes `color_names` et comptage via `Counter`.
  - Barres des 15 couleurs les plus fréquentes pour identifier les dominantes globales (ex. `bleu`, `vert`, `orange`).

### 3. Préférences utilisateurs
Deux types de graphiques :
- **Barres empilées – Couleurs favorites par utilisateur**
  - Construction d’une matrice binaire (one-hot) indiquant, pour chaque utilisateur, quelles couleurs figurent dans `favorite_colors`.
  - Représentation en barres empilées par utilisateur, chaque couleur ayant sa propre couleur de barre.
  - Donne un aperçu visuel des palettes préférées de chaque profil.
- **Comparaison des orientations et tailles favorites**
  - Deux diagrammes en barres :
    - `favorite_orientation` (paysage, portrait, …) par nombre d’utilisateurs.
    - `favorite_size` (vignette, moyenne, grande) par nombre d’utilisateurs.
  - Permet de comparer qualitativement les préférences structurelles des utilisateurs (forme et taille des images aimées).

### 4. Analyse des tags
Une visualisation principale :
- **Barres – Tags les plus fréquents**
  - Aplatissement de toutes les listes de `tags` dans `labels_df`.
  - Comptage via `Counter`, affichage du top 20 en barres horizontales.
  - Objectif : mettre en évidence les thématiques les plus représentées dans la collection (ex. `nature`, `mer`, `ville`, etc.).

## Résultat
- Au total, plus de **6 visualisations distinctes** sont produites dans le notebook :
  - 3 pour les statistiques d’images, 2 pour les couleurs, 2 pour les préférences utilisateurs et 1 pour les tags (certains combinés en subplots).
- Chaque graphique possède :
  - Un **titre** explicite.
  - Des axes nommés (`xlabel`, `ylabel`) lorsque pertinent.
  - Une **légende** pour les graphiques empilés ou multi-séries.

## Choix techniques et limites
- Les visualisations s’appuient sur **matplotlib** et **pandas** pour simplifier le regroupement et l’agrégation.
- Les tags sont affichés bruts ; un nettoyage supplémentaire (stopwords, regroupement sémantique) améliorerait la lisibilité pour de grands jeux de données.
- La palette de couleurs affichée est limitée à un sous-échantillon pour rester lisible ; une interface interactive pourrait permettre d’explorer davantage.

## Pistes d’amélioration
- Ajouter un nuage de mots pour les tags les plus fréquents.
- Introduire des visualisations interactives (ex. `plotly`) pour explorer dynamiquement les préférences.
- Lier ces visualisations au futur système de recommandation (Tâche 5), par exemple en mettant en avant les images recommandées pour un utilisateur donné.

