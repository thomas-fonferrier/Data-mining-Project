# Rapport Tâche 3 : Analyse des données

## Objectif
Construire des profils de préférences pour plusieurs utilisateurs simulés à partir des annotations d’images (Tâche 2), puis analyser les tendances globales (couleurs, tags, groupes d’utilisateurs).

## Données d’entrée
- `data/images_labels.json` : annotations par image issues de la Tâche 2, au format :
  - `predominant_colors` (liste de couleurs RGB)
  - `color_names` (noms de couleurs)
  - `orientation`
  - `size_category`
  - `tags`

## Étapes d’implémentation dans `project.ipynb`

### 1. Simulation d’utilisateurs et de favoris
Fonction `simulate_users(labels, n_users=5, min_fav=10, max_fav=20)` :
- Récupère la liste des images annotées et l’ensemble des tags (hors `auteur:`).
- Crée au moins 5 utilisateurs (`utilisateur_001`, …).
- Pour chaque utilisateur :
  - Tire aléatoirement un `preferred_topic` parmi les tags disponibles.
  - Sélectionne 10 à 20 images favorites :
    - ≈ 60 % des favoris biaisés vers le `preferred_topic`.
    - Le reste choisi aléatoirement dans le catalogue.
- Sortie : une liste de dictionnaires contenant `user_id`, `preferred_topic` et `favorite_images`.

### 2. Construction des profils utilisateurs
Fonction `build_user_profiles(users, labels)` :
- Pour chaque utilisateur, récupère les annotations de ses images favorites.
- Agrège les informations pour construire un profil :
  - **Couleurs favorites** : top 5 des `color_names` les plus fréquentes.
  - **Orientation favorite** : modalité la plus fréquente parmi `paysage`, `portrait`, `carre`, `inconnu`.
  - **Taille favorite** : modalité la plus fréquente parmi `vignette`, `moyenne`, `grande`, `inconnu`.
  - **Tags favoris** : top 10 des tags les plus fréquents.
  - **Images favorites** : liste exacte des fichiers sélectionnés.
- Sortie : liste de `profiles`, sérialisée ensuite dans `data/users.json`.

Structure d’un profil (conforme à la consigne) :
```json
{
  "user_id": "utilisateur_001",
  "preferred_topic": "nature",
  "favorite_colors": ["bleu", "vert"],
  "favorite_orientation": "paysage",
  "favorite_size": "moyenne",
  "favorite_tags": ["nature", "mer", "ciel", "..."],
  "favorite_images": ["1.jpg", "7.jpg", "..."]
}
```

### 3. Analyse des tendances globales
Fonction `analyze_preferences(profiles)` :
- Construit un DataFrame `pandas` à partir des profils pour faciliter les analyses.
- **Popularité globale des couleurs** :
  - Aplatissement de toutes les listes `favorite_colors`.
  - Comptage avec `Counter` et affichage du top 10.
- **Popularité globale des tags** :
  - Aplatissement de toutes les listes `favorite_tags`.
  - Comptage avec `Counter` et affichage du top 10.
- **Clustering des utilisateurs** (optionnel mais implémenté) :
  - Construction de vecteurs one-hot par utilisateur sur l’espace des couleurs observées.
  - Application de KMeans (2 à 3 clusters selon le nombre d’utilisateurs).
  - Affichage, pour chaque cluster, de la liste des `user_id` qui y appartiennent.

La fonction retourne `color_counts` et `tag_counts` pour une éventuelle réutilisation dans les tâches suivantes (visualisation, recommandation).

### 4. Sauvegarde des résultats
- Les profils utilisateurs sont écrits dans `data/users.json` en UTF-8, JSON indenté.
- Les principales statistiques (couleurs et tags les plus populaires, clusters) sont affichées dans la sortie du notebook.

## Résultat
- **Fichier généré** : `data/users.json` avec un profil structuré pour chaque utilisateur simulé.
- **Analyses produites** :
  - Top couleurs favorites globales.
  - Top tags favoris globaux.
  - Regroupement d’utilisateurs par similarité de couleurs (clusters).

## Choix techniques et limites
- Les utilisateurs sont **simulés** avec un biais thématique simple (topic issu des tags), ce qui suffit pour tester la chaîne d’analyse mais ne représente pas des comportements réels.
- Les profils sont basés sur les annotations de Tâche 2 ; la qualité des résultats dépend donc directement de la qualité des tags et des couleurs extraits.
- Le clustering utilise uniquement les couleurs favorites (one-hot binaire), ce qui capture bien des préférences grossières mais ignore orientation, taille et tags.

## Pistes d’amélioration
- Affiner la simulation des utilisateurs (profils plus variés, recouvrements partiels de préférences, bruit aléatoire).
- Étendre le vecteur de caractéristiques des utilisateurs (inclure orientation, taille, tags) pour un clustering plus riche.
- Ajouter des visualisations dédiées à la Tâche 3 (cartes de chaleur, matrices de co-occurrence tags × couleurs, etc.).

