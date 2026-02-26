# Rapport Tâche 2 : Étiquetage et annotation

## Objectif
Enrichir les images téléchargées avec des annotations structurées (couleurs prédominantes, orientation, catégorie de taille, tags) et produire un fichier JSON `data/images_labels.json` conforme aux exigences de la Tâche 2 du projet.

## Données d’entrée
- Métadonnées brutes : `data/images_metadata.json` (créé en Tâche 1) contenant pour chaque image : nom de fichier, dimensions, format, taille, URL source, description, auteur, etc.
- Images téléchargées : dossier `images/`.

## Implémentation dans `project.ipynb`
Les ajouts/ajustements se trouvent sous la section **Tâche 2** :

- **Chargement robuste des métadonnées** : lecture sécurisée du JSON (tolérance aux fichiers vides ou invalides).
- **Extraction des couleurs dominantes** : `extract_dominant_colors` utilise KMeans (5 couleurs par défaut), triées par fréquence. Chaque couleur est convertie en RGB entier.
- **Nommage des couleurs** : correspondance au plus proche dans une palette de base (`rouge`, `bleu`, `vert`, `jaune`, etc.) via distance euclidienne.
- **Orientation** : `paysage`, `portrait`, ou `carre` (tolérance de 5 %) d’après largeur/hauteur.
- **Catégorie de taille** : `vignette` (<500px), `moyenne` (500–1500px), `grande` (>1500px) selon le côté le plus long.
- **Tags** : combinaison de la requête (topic), des mots-clés de description (filtrés >3 caractères), de l’auteur (`auteur:<username>`) et des noms de couleurs.
- **Sauvegarde** : toutes les annotations sont écrites dans `data/images_labels.json` (dict indexé par nom de fichier) avec les clés :
  - `predominant_colors` (liste RGB)
  - `color_names`
  - `orientation`
  - `size_category`
  - `tags`

## Résultat
- Exécution de la cellule : génération des annotations pour les images présentes et écriture dans `data/images_labels.json`.
- Affichage d’un aperçu (3 premières images) dans le notebook pour validation rapide.

## Choix techniques et limites
- **Couleurs** : palette volontairement réduite pour rester lisible et sans dépendance externe. Peut être raffinée avec une liste CSS plus riche si besoin.
- **Tags** : approche automatique simple (description + topic + auteur + couleurs). Pour une meilleure qualité, on pourra ajouter un dictionnaire de stopwords ou des tags sources de l’API.
- **Dimensions** : priorité aux métadonnées stockées ; relecture de l’image si manquantes.
- **Résilience** : saut des images manquantes et des JSON invalides, sans bloquer le pipeline.

## Pistes d’amélioration
- Étendre le mapping des noms de couleurs ou utiliser un nuancier complet.
- Ajouter un nettoyage de texte plus poussé (stopwords, lemmatisation) pour les tags.
- Stocker la confiance/poids de chaque couleur (fréquence normalisée).
- Enrichir les tags avec les EXIF (modèle d’appareil) et/ou les mots-clés de la source (API).

