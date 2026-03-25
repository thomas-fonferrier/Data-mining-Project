Summary Report — Image Recommendation System
1. Introduction
The objective of this project was to build, end-to-end, an image recommendation system: collect a dataset of royalty-free images, automatically annotate them, analyze simulated user preferences, and then suggest new images to each user based on their tastes.

The guiding idea was not only to produce a “black box” system that recommends without explanation, but to be able to provide a reason — even if imperfect — for each suggestion.

To achieve this, the pipeline is structured into five main steps:

Automated image collection via the Unsplash API,
Annotation of each image (colors, orientation, size, tags),
Simulation of user profiles and analysis of their preferences,
Data visualization,
Personalized recommendation using an SVM classifier.
2. Data Collection
Source and License
The 100 images were automatically collected via the Unsplash API (free, requires an access key). Unsplash provides images under a free-to-use license (Unsplash License), allowing their use in a non-commercial project.

The query used was "France", resulting in a thematically consistent dataset — landscapes, monuments, Parisian streets — but also intentionally limited, in order to test the system’s robustness on a homogeneous dataset.

Download Automation
The script iterates over 10 pages of 10 results via the /search/photos endpoint. For each image, it:

downloads the file in regular resolution (~1080px wide),
assigns it a numeric name (1.jpg to 100.jpg),
extracts and stores metadata in data/images_metadata.json.
Stored Metadata
For each image, the JSON file contains:

Field	Description
file_name	file name
dimensions	width and height (pixels)
file_format	JPEG in all cases
size_ko	file size (KB)
url_source	original Unsplash URL
informations_licence	“Not specified” due to lack of structured API data
exif	EXIF data if available (often null on Unsplash)
created_at	publication date
description	caption provided by the photographer
username	Unsplash username
3. Methodology
3.1. Labeling and Annotation (Task 2)
Each image was automatically annotated using Python:

Dominant colors. The image is resized to 256px width, pixels are flattened into an (n, 3) array, and KMeans (k = 5) is applied. The 5 centroids represent dominant colors, sorted by frequency. Each RGB color is then mapped to a readable name (black, white, gray, red, green, blue, yellow, orange, purple, pink, brown, turquoise) using Euclidean distance in RGB space to a reference palette.

Orientation. Computed from the width/height ratio, with a ±5% tolerance for “square” images.

Size category. Based on the largest side: thumbnail (< 500 px), medium (500–1500 px), large (> 1500 px).

Tags. Built by combining:

the search theme (france),
words extracted from the Unsplash description (filtered > 3 characters),
the photographer’s username (prefixed author:),
color names.
Annotations are stored in data/images_labels.json.

3.2. User Profiles (Task 3)
Five simulated users were generated (user_001 to user_005), each having favorited between 10 and 20 images.

To introduce preference diversity, each user is biased toward a specific tag (randomly selected among available tags): 60% of their favorites come from images with this tag, while the remaining 40% are randomly selected.

Extracted profile:

Field	Method
favorite_colors	top 5 most frequent color names among favorites
favorite_orientation	most frequent category
favorite_size	most frequent category
favorite_tags	top 10 most frequent tags
Profiles are saved in data/users.json.

Global trend analysis. Using Counter (collections) and pandas, the most popular colors and tags across all users were computed. A KMeans clustering on binary vectors of favorite colors grouped users into 3 clusters, revealing that 3 users share similar preferences (gray/black/white), while the other 2 have more distinctive profiles.

3.3. Recommendation System (Task 5 — Option A: SVM)
The chosen approach is content-based filtering using a linear SVM (LinearSVC).

Image representation. Each image is transformed into a text document (tags + colors + orientation + size). A CountVectorizer converts this corpus into a binary matrix (presence/absence of tokens).

Training. For each user, a binary SVM is trained:

class 1 = favorite images,
class 0 = other images.
The model learns a linear boundary separating images that match the user’s taste from those that do not.

Recommendation. The decision_function is used to score all non-favorited images. The highest-scoring ones are recommended.

Explanation of recommendations. For each recommended image, tokens with high positive SVM weights are identified: these are the features driving the score toward “favorite”. These tokens are displayed with their weights, providing a minimal explanation.

4. Results
Key Visualizations
Four groups of visualizations were produced (Task 4):

Collection statistics: orientation (almost 100% landscape, consistent with the orientation=landscape filter), size (mostly “medium” between 500 and 1500 px), format (100% JPEG).

Color palette: the France dataset is dominated by gray, white, brown, and green tones — typical of pavements, building facades, and vegetation.

User preferences: stacked bars show that most users share a large base of common colors (gray, black, white), making profile differentiation more difficult.

Most frequent tags: france, gray, white, black, green dominate — showing that the collection theme heavily influences generated tags.

Recommendation Quality
For the 5 tested users, the system consistently returns 5 non-favorited images, satisfying the basic constraint. Recommendations for user_004 (biased toward paris / orange) are qualitatively relevant: images of museums, the Eiffel Tower, with orange and white tones appear.

Recommendations for other users are more debatable, as the dataset is too homogeneous for the SVM to extract sufficiently discriminative signals.

5. Limitations and Future Work
Lack of Interpretability
The main limitation is the difficulty of producing clear and meaningful explanations. When the model outputs kmile_ch (0.71) as the main reason, it actually indicates that the photographer’s username is the most discriminative feature. Technically correct — but not useful for a human user.

This issue comes from:

CountVectorizer splitting composite tokens (author:kmile_ch → author + kmile_ch),
Unsplash descriptions containing proper names and handles that do not describe visual features.
Possible improvements:

Explicitly filter tokens starting with author: when generating explanations,
Build an average user profile (colors, orientation, size) and express explanations as a distance to the profile (“recommended because it matches your usual preferences: landscape orientation and dominant blue color”).
Lack of Dataset Diversity
All images are tied to the query France, resulting in very similar tags and colors. This weakens the SVM, which struggles to find clear decision boundaries.

Improvement Directions
Combine SVM and KMeans. Use clustering to group images into visual themes. The SVM selects images, while clusters provide more natural explanations (“recommended because it belongs to the urban architecture group you tend to like”).

Enrich features. Add visual features (color ratios, contrast, brightness) to improve model relevance.

Use TF-IDF instead of CountVectorizer. This would reduce the weight of very frequent tokens (france, landscape) and emphasize rarer, more discriminative ones.

6. Conclusion
This project enabled the construction of a complete image recommendation pipeline, from automated collection to personalized suggestions. Each step builds on the previous one: metadata feeds annotation, which feeds profiles, which feed the model.

The most important takeaway is that recommendation quality depends heavily on the quality and diversity of the data. With only 100 images tied to a single theme, user preferences become very similar, limiting the SVM’s effectiveness.

From a technical perspective, a linear SVM is a solid choice: fast to train, interpretable via weights, and flexible enough to adapt to each user. The identified improvements (filtering non-visual tokens, profile-based explanations, combining with KMeans) could elevate this prototype into a production-ready system.

References
Unsplash API: https://unsplash.com/developers
Scikit-learn: LinearSVC, CountVectorizer, KMeans — https://scikit-learn.org
Pillow (PIL): color extraction and EXIF — https://pillow.readthedocs.io
Pandas: user profile analysis — https://pandas.pydata.org
Matplotlib: visualizations — https://matplotlib.org
