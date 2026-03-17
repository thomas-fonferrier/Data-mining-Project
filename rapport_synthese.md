# Rapport de synthèse — Système de recommandation d'images

---

## 1. Introduction

Ce projet avait pour objectif de construire, de bout en bout, un **système de recommandation d'images** : collecter un corpus d'images libres de droits, les annoter automatiquement, analyser les préférences d'utilisateurs simulés, puis suggérer de nouvelles images à chacun d'eux en fonction de ce qu'ils aiment.

L'idée directrice était de ne pas seulement produire un système « boîte noire » qui recommande sans expliquer, mais d'être capable de donner une raison — même imparfaite — à chaque suggestion.

Pour cela, notre pipeline s'articule en cinq grandes étapes :

1. Collecte automatisée d'images via l'API Unsplash,
2. Annotation de chaque image (couleurs, orientation, taille, tags),
3. Simulation de profils utilisateurs et analyse de leurs tendances,
4. Visualisation des données,
5. Recommandation personnalisée par un classificateur SVM.

---

## 2. Collecte de données

### Source et licence

Les 100 images ont été collectées automatiquement via l'**API Unsplash** (gratuite, nécessite une clé d'accès). Unsplash propose des images sous **licence libre d'utilisation** (Unsplash License), ce qui autorise leur exploitation dans un projet non commercial.

La requête utilisée était `"France"`, ce qui donne un corpus thématiquement cohérent — paysages, monuments, rues parisiennes — mais aussi volontairement limité, pour tester la robustesse du système face à un jeu de données homogène.

### Automatisation du téléchargement

Le script parcourt 10 pages de 10 résultats via l'endpoint `/search/photos`. Pour chaque image, il :

- télécharge le fichier en résolution `regular` (≈ 1080px de large),
- lui attribue un nom numérique (`1.jpg` à `100.jpg`),
- extrait et stocke ses métadonnées dans `data/images_metadata.json`.

### Métadonnées stockées

Pour chaque image, le fichier JSON contient :

| Champ | Description |
|---|---|
| `file_name` | nom du fichier |
| `dimensions` | largeur et hauteur (pixels) |
| `file_format` | JPEG dans tous les cas ici |
| `size_ko` | taille sur disque (Ko) |
| `url_source` | URL Unsplash originale |
| `informations_licence` | «Non spécifié» faute de donnée structurée dans l'API |
| `exif` | données EXIF si disponibles (souvent `null` sur Unsplash) |
| `created_at` | date de publication |
| `description` | légende fournie par le photographe |
| `username` | pseudonyme Unsplash |

---

## 3. Méthodologie

### 3.1. Étiquetage et annotation (Tâche 2)

Chaque image a été annotée automatiquement à l'aide de Python :

**Couleurs prédominantes.** On redimensionne l'image à 256 px de large, on aplatit les pixels en un tableau `(n, 3)` et on applique un **KMeans** (k = 5). Les 5 centroïdes représentent les couleurs dominantes, triées par fréquence. Chaque couleur RGB est ensuite traduite en nom lisible (noir, blanc, gris, rouge, vert, bleu, jaune, orange, violet, rose, marron, turquoise) par distance euclidienne dans l'espace RGB à une palette de référence.

**Orientation.** Calculée à partir du ratio largeur/hauteur, avec une tolérance de ± 5 % pour les images « carrées ».

**Catégorie de taille.** Basée sur le plus grand côté : vignette (< 500 px), moyenne (500–1500 px), grande (> 1500 px).

**Tags.** Construits en combinant :
- le thème de la recherche (ici `france`),
- les mots extraits de la description Unsplash (filtrés > 3 caractères),
- le pseudonyme du photographe (préfixé `auteur:`),
- les noms de couleurs.

Les annotations sont stockées dans `data/images_labels.json`.

### 3.2. Profils utilisateurs (Tâche 3)

Cinq utilisateurs simulés ont été générés (`utilisateur_001` à `utilisateur_005`), chacun ayant favoritisé entre 10 et 20 images.

Pour introduire une diversité de préférences, chaque utilisateur est biaisé vers un tag particulier (choisi aléatoirement parmi les tags disponibles) : 60 % de ses favoris proviennent d'images ayant ce tag, les 40 % restants sont pris aléatoirement dans le catalogue.

**Profil extrait :**

| Champ | Méthode |
|---|---|
| `favorite_colors` | top 5 des `color_names` les plus fréquentes parmi les favoris |
| `favorite_orientation` | modalité la plus fréquente |
| `favorite_size` | modalité la plus fréquente |
| `favorite_tags` | top 10 des tags les plus récurrents |

Les profils sont sauvegardés dans `data/users.json`.

**Analyse des tendances globales.** À l'aide de `Counter` (collections) et `pandas`, on a calculé les couleurs et tags les plus populaires tous utilisateurs confondus. Un **KMeans** sur des vecteurs binaires de couleurs favorites a permis de regrouper les utilisateurs en 3 clusters, révélant que 3 d'entre eux partagent des préférences proches (gris/noir/blanc) tandis que 2 autres ont des profils plus distinctifs.

### 3.3. Système de recommandation (Tâche 5 — Option A : SVM)

L'approche choisie est un **filtrage basé sur le contenu** avec un **SVM linéaire** (`LinearSVC`).

**Représentation des images.** Chaque image est transformée en un document texte (tags + couleurs + orientation + taille). Un `CountVectorizer` transforme ce corpus en une matrice binaire (présence/absence de chaque token).

**Entraînement.** Pour chaque utilisateur, on entraîne un SVM binaire :
- classe 1 = images favorites,
- classe 0 = autres images.

Le modèle apprend ainsi une frontière linéaire séparant les images « dans le goût de l'utilisateur » de celles qui ne le sont pas.

**Recommandation.** On utilise la `decision_function` pour scorer toutes les images non encore favorites. Les images avec le score le plus élevé sont proposées.

**Explication des recommandations.** Pour chaque image recommandée, on identifie les tokens présents dont le poids SVM est positif et élevé : ce sont les caractéristiques qui « tirent » effectivement le score vers « favori ». Ces tokens sont affichés avec leur poids, permettant une explication minimale.

---

## 4. Résultats

### Visualisations clés

Quatre groupes de visualisations ont été produits (Tâche 4) :

1. **Statistiques de la collection** : orientation (quasiment 100 % paysage, cohérent avec le filtre `orientation=landscape` à la collecte), taille (majorité « moyenne » entre 500 et 1500 px), format (100 % JPEG).

2. **Palette de couleurs** : la collection France est dominée par les tons gris, blanc, marron et vert — caractéristiques des pavés, façades et végétation.

3. **Préférences utilisateurs** : les barres empilées montrent que la plupart des utilisateurs partagent une large base de couleurs communes (gris, noir, blanc), ce qui rend la différenciation des profils plus difficile.

4. **Tags les plus fréquents** : `france`, `gris`, `blanc`, `noir`, `vert` arrivent en tête — signe que la thématique de collecte domine très largement les tags générés.

### Qualité des recommandations

Sur les 5 utilisateurs testés, le système retourne systématiquement 5 images non favorites, ce qui valide la contrainte de base. Les recommandations pour `utilisateur_004` (biaisé vers `paris` / `orange`) sont qualitativement pertinentes : on y retrouve des images de musées, de la tour Eiffel, avec les couleurs orange et blanc. Les recommandations pour d'autres utilisateurs sont plus discutables, le corpus étant trop homogène pour que le SVM dispose de suffisamment de signal discriminant.

---

## 5. Limitations et travaux futurs

### Opacité des raisons

La limite la plus notable de notre système est la difficulté à produire des **raisons lisibles et vraiment pertinentes**. Lorsque le modèle affiche `kmile_ch (0.71)` comme principale raison d'une recommandation, il dit en réalité que le pseudonyme du photographe est le signe le plus discriminant dans ses données. Techniquement c'est juste — l'utilisateur a peut-être aimé plusieurs photos de cet auteur — mais c'est peu utile et déconcertant pour un utilisateur humain.

Le problème vient de deux sources :
- Le `CountVectorizer` fragmente les tokens composés (`auteur:kmile_ch` devient `auteur` + `kmile_ch`), et les noms propres / handles se retrouvent propulsés en première ligne.
- Les descriptions Unsplash contiennent des noms propres, des lieux ou des handles qui ne disent rien des caractéristiques visuelles de l'image.

**Améliorations envisageables :**
- Filtrer explicitement les tokens qui commencent par `auteur:` lors de la construction de l'explication.
- Construire un **profil moyen de l'utilisateur** (couleurs, orientation, taille les plus représentées dans ses favoris) et exprimer la raison comme une **distance au profil** : « recommandée car orientation paysage et couleur dominante bleue, proches de vos favoris habituels ».

### Manque de diversité du corpus

Toutes les images étant liées à la requête `France`, les tags et couleurs sont très proches entre images. Cela affaiblit le SVM, qui peine à trouver des frontières claires entre favoris et non favoris.

### Pistes d'amélioration

**Combiner SVM et KMeans.** On pourrait utiliser le clustering (KMeans sur les vecteurs d'images) pour regrouper les images en thèmes visuels cohérents. Le SVM désignerait une image à recommander, et le cluster de cette image fournirait une raison plus naturelle : « recommandée car elle appartient au groupe *architecture urbaine*, que vous avez tendance à apprécier ». Cela enrichirait à la fois la pertinence et l'explicabilité.

**Enrichir les features.** Ajouter des features plus visuelles (ratios de couleurs dans l'image, contraste, luminosité moyenne) permettrait au SVM de travailler sur de vrais attributs visuels plutôt que sur des mots-clés textuels.

**TF-IDF au lieu de CountVectorizer.** Remplacer le comptage brut par un TF-IDF réduirait le poids des tokens très fréquents (présents dans quasiment toutes les images, comme `france` ou `paysage`) au profit de tokens plus rares et donc plus discriminants.

---

## 6. Conclusion

Ce projet nous a permis de construire un pipeline complet de recommandation d'images, depuis la collecte automatisée jusqu'à la suggestion personnalisée. Chaque tâche s'appuie sur la précédente : les métadonnées alimentent l'annotation, qui alimente les profils, qui alimentent le modèle.

La partie la plus instructive a été de réaliser que la qualité d'un système de recommandation dépend avant tout de la **qualité et de la diversité des données**. Avec 100 images toutes liées au même thème, les préférences des utilisateurs simulés se ressemblent beaucoup, et le SVM ne dispose pas d'assez de signal pour se distinguer vraiment.

Sur le plan technique, l'utilisation d'un SVM linéaire est un choix solide : rapide à entraîner, interprétable via ses poids, et suffisamment souple pour s'adapter à chaque utilisateur. Les pistes d'amélioration identifiées (filtrage des tokens non visuels, profil utilisateur pour les explications, combinaison avec KMeans) permettraient de faire passer ce prototype au niveau d'un système utilisable en conditions réelles.

---

## Références

- Unsplash API : https://unsplash.com/developers
- Scikit-learn : `LinearSVC`, `CountVectorizer`, `KMeans` — https://scikit-learn.org
- Pillow (PIL) : extraction de couleurs et EXIF — https://pillow.readthedocs.io
- Pandas : analyse des profils utilisateurs — https://pandas.pydata.org
- Matplotlib : visualisations — https://matplotlib.org
