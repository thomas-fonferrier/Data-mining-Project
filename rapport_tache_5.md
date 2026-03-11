# Rapport Tâche 5 : Système de recommandation (Option A – SVM basé sur le contenu)

## Objectif

Implémenter un système de recommandation d’images qui, pour un utilisateur donné, propose de nouvelles images **similaires à celles qu’il a déjà aimées**, en se basant uniquement sur le **contenu** (métadonnées enrichies) et en utilisant un **classificateur SVM**.

Dans `project.ipynb`, cette tâche est implémentée dans une nouvelle cellule Python intitulée :

> `# Tâche 5 : Système de recommandation (Option A - SVM)`

Elle s’appuie sur les sorties des tâches précédentes :
- `data/images_labels.json` (Tâche 2) : annotations par image (couleurs, orientation, taille, tags).
- `data/users.json` (Tâche 3) : profils utilisateurs avec leurs `favorite_images`.

---

## Données et prétraitement

### Chargement des données

Au début de la cellule Tâche 5 :

- Les annotations d’images sont chargées :
  - `images_labels = json.load(open("data/images_labels.json"))`
- Les profils utilisateurs sont chargés :
  - `user_profiles = json.load(open("data/users.json"))`

Ces deux structures sont ensuite utilisées dans toutes les fonctions de recommandation.

### Construction d’un corpus texte par image

Fonction :

```python
def _build_image_corpus() -> Tuple[List[str], List[str]]:
    ...
```

But :
- Transformer chaque image en une **phrase de mots-clés** à partir de ses métadonnées enrichies :
  - `tags` (sujet, description, auteur, couleurs intégrées dans les tags),
  - `color_names` (couleurs prédominantes),
  - `orientation` (paysage, portrait, carré…),
  - `size_category` (vignette, moyenne, grande…).

Pour chaque image :
- On concatène ces éléments en une chaîne de texte, séparée par des espaces.
- On remplit :
  - `image_ids` : liste des noms de fichiers (`"1.jpg"`, `"2.jpg"`, etc.).
  - `corpus` : liste des documents texte correspondants.

Ce corpus est ensuite vectorisé par un **CountVectorizer**, ce qui permet d’avoir une représentation numérique simple des caractéristiques de chaque image (sac de mots).

---

## Modèle de classification : SVM binaire

### Entraînement par utilisateur

Fonction :

```python
def _train_user_svm(user_id: str) -> Tuple[LinearSVC, np.ndarray, List[str]]:
    ...
```

Étapes :

1. **Vectorisation des images**
   - On appelle `_build_image_corpus()` pour obtenir `image_ids` et `corpus`.
   - `CountVectorizer()` est utilisé pour transformer `corpus` en matrice `X` (documents × mots).
   - Chaque ligne de `X` représente une image via ses mots-clés (tags, couleurs, etc.).

2. **Étiquettes Favori / Non favori**
   - On récupère le profil de l’utilisateur (`user_profiles`) pour avoir la liste `favorite_images`.
   - On crée un vecteur `y` :
     - `y[i] = 1` si `image_ids[i]` est dans les favoris de l’utilisateur,
     - `y[i] = 0` sinon.

3. **Contrôle des classes**
   - Le SVM a besoin de **deux classes** différentes (au moins un favori et une non-favori).
   - On calcule :

     ```python
     n_pos = int(y.sum())
     n_neg = int(len(y) - n_pos)
     if n_pos == 0 or n_neg == 0:
         raise ValueError("L'utilisateur doit avoir au moins une image favorite et une non favorite pour entraîner le modèle.")
     ```

   - Si ce n’est pas respecté, on arrête proprement avec une erreur explicite.

4. **Entraînement du SVM**
   - On utilise un SVM linéaire :

     ```python
     clf = LinearSVC()
     clf.fit(X, y)
     ```

   - Le modèle apprend une frontière linéaire séparant les images favorites des non favorites.

5. **Retour**
   - La fonction renvoie :
     - le modèle entraîné `clf`,
     - la matrice de caractéristiques `X` (pour toutes les images),
     - la liste `image_ids` (ordre compatible avec `X`).

---

## Fonction principale de recommandation

### Signature

```python
def recommend_images(user_id: str, n_recommendations: int = 5) -> List[Tuple[str, str]]:
    """
    Recommander des images pour un utilisateur.

    Args:
        user_id: L'utilisateur pour lequel recommander
        n_recommendations: Nombre d'images à recommander

    Returns:
        Liste de tuples (nom_fichier_image, raison)
    """
```

### Logique détaillée

1. **Entraînement du modèle pour l’utilisateur**

```python
try:
    clf, X, image_ids = _train_user_svm(user_id)
except ValueError as e:
    print(f"Impossible d'entraîner le modèle pour {user_id} : {e}")
    return []
```

- Si l’utilisateur n’a pas assez de diversité (pas de non-favoris ou pas de favoris), la fonction affiche un message et retourne une liste vide.

2. **Exclusion des favoris existants**

```python
profile = next(p for p in user_profiles if p["user_id"] == user_id)
fav_set = set(profile.get("favorite_images", []))
```

- On garde les favoris dans un `set` pour ne pas les proposer à nouveau.

3. **Scorage de toutes les images**

```python
scores = clf.decision_function(X)
ranked = sorted(zip(image_ids, scores), key=lambda t: t[1], reverse=True)
```

- `decision_function` du SVM renvoie un score de distance à l’hyperplan :
  - Score élevé → image plus probable d’être un favori.
- On trie toutes les images par score décroissant.

4. **Sélection des recommandations**

Pour chaque image triée :
- On saute les images déjà favorites (`if img_id in fav_set: continue`).
- On récupère les métadonnées de l’image :

  ```python
  meta = images_labels[img_id]
  top_colors = meta.get("color_names", [])[:2]
  orientation = meta.get("orientation")
  size_cat = meta.get("size_category")
  ```

- On compose une **raison textuelle** simple, lisible, en utilisant les informations pertinentes :

  ```python
  reason_parts = []
  if top_colors:
      reason_parts.append(f"couleurs {', '.join(top_colors)}")
  if orientation:
      reason_parts.append(f"orientation {orientation}")
  if size_cat:
      reason_parts.append(f"taille {size_cat}")
  reason = " et ".join(reason_parts) if reason_parts else "car elle ressemble aux images favorites"
  ```

- On ajoute `(img_id, reason)` à la liste de résultats.
- On s’arrête lorsque `n_recommendations` images ont été collectées.

5. **Retour**

La fonction retourne une liste :

```python
[
  ("1.jpg", "couleurs gris, vert et orientation paysage"),
  ("7.jpg", "couleurs bleu, orange et taille moyenne"),
  ...
]
```

Chaque entrée contient :
- le nom de fichier de l’image recommandée,
- une explication courte basée sur les caractéristiques (couleurs / orientation / taille).

---

## Exemple d’utilisation

En fin de cellule, un exemple est fourni :

```python
example_user = user_profiles[0]["user_id"] if user_profiles else "utilisateur_001"
reco = recommend_images(example_user, n_recommendations=5)
print(f"Recommandations pour {example_user} :")
for img, why in reco:
    print(f"- {img} -> {why}")
```

- On prend le premier utilisateur existant (ou `utilisateur_001` par défaut).
- On demande 5 recommandations.
- On affiche pour chaque image recommandée une phrase du type :
  - `- 12.jpg -> couleurs gris, vert et orientation paysage`

---

## Choix techniques et limites

- **Représentation des images** : on utilise un sac de mots simple (tags, couleurs, orientation, taille) via `CountVectorizer`. C’est minimaliste mais cohérent avec les métadonnées disponibles.
- **Modèle** : `LinearSVC` (SVM linéaire) est rapide et adapté à la classification binaire Favori / Non favori.
- **Par utilisateur** : un modèle SVM est entraîné **indépendamment pour chaque utilisateur**, ce qui capture ses préférences personnelles.
- **Contraintes de classes** : pour éviter les erreurs de scikit-learn, on vérifie qu’il existe au moins une image favorite et une non favorite. Sinon, on ne propose pas de recommandations pour cet utilisateur.
- **Explications** : les raisons affichées sont simples et directement tirées des métadonnées (couleurs, orientation, taille). On pourrait les enrichir en intégrant davantage d’informations (tags sémantiques, EXIF, etc.).

---

## Pistes d’amélioration

- Utiliser un **TF-IDF** au lieu du CountVectorizer brut, pour donner plus de poids aux termes discriminants.
- Introduire une pondération différente pour les caractéristiques (par exemple, couleurs plus importantes que taille, etc.).
- Ajouter une gestion plus élaborée des cas limites :
  - si peu de favoris, compléter avec du contenu proche sans entraînement supervisé.
- Combiner cette approche de filtrage par contenu avec une logique de **clustering** (Tâche 3) ou de profils utilisateurs pour créer une approche hybride.

