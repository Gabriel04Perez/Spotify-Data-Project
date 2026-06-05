# Do Popular Artists Make Popular Music?
By: Gabriel Isaiah Perez

---

## Introduction

Spotify is one of the world's largest music streaming platforms, hosting tens of millions of tracks across every conceivable genre. Two key metrics define a track's success: the **track popularity score** (a 0–100 integer reflecting recent streaming activity) and the **artist follower count** (the cumulative audience an artist has built over time). But are these two metrics actually related? Does being a famous artist guarantee that your music will be popular?

This project investigates the following central question:

> **Do tracks from artists with more followers tend to be more popular than tracks from artists with fewer followers?**

This question matters to listeners, labels, and independent artists alike. If follower count strongly predicts track popularity, it suggests platform success is self-reinforcing — the famous get more famous regardless of the music itself. If follower count does *not* predict popularity, it implies that audio characteristics or release timing play a more decisive role, which is encouraging for smaller artists.

I analyzed two datasets joined on artist name:

- **`music_tracks`**: 114,000 rows × 21 columns — one row per Spotify track, with Spotify-computed audio features.
- **`artists`**: 1.2 million rows × 5 columns — one row per artist, with follower counts and genre tags.

I focused on five musically distinct genres (**hip-hop**, **country**, **gospel**, **tango**, and **latin**) which together span a wide range of audio characteristics and make for a meaningful cross genre comparison.

The columns most relevant to our central question are:

| Column | Description |
|--------|-------------|
| `popularity` | Track's Spotify popularity score (0–100); reflects recent and frequent streaming |
| `artist_followers` | Number of Spotify followers for the track's primary artist |
| `track_genre` | Musical genre (hip-hop, country, gospel, tango, latin) |
| `danceability` | How suitable for dancing (0 = least, 1 = most) |
| `energy` | Perceptual measure of intensity and activity (0 = least, 1 = most) |
| `acousticness` | Confidence that the track is acoustic (0 = least, 1 = most) |
| `speechiness` | Presence of spoken words: > 0.33 indicates rap or vocal-heavy content |
| `valence` | Musical positiveness (0 = negative, 1 = positive) |
| `tempo` | Estimated tempo in BPM |
| `explicit` | Whether the track contains explicit content (`True`/`False`) |
| `duration_ms` | Track duration in milliseconds |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

I performed the following cleaning steps before analysis:

1. **Parsed `release_date` → `release_year`**: The raw column mixed three date formats (YYYY-MM-DD, YYYY-MM, YYYY). Extracted only the four digit year and dropped the original column.

2. **Filtered to five genres**: I retained only rows where `track_genre` is one of hip-hop, country, gospel, tango, or latin. These five are musically distinct across all key audio dimensions. Hip-hop has high speechiness and low acousticness, tango has very high acousticness and a distinctive tempo, gospel is choir driven with high acousticness, country is vocal and acoustic, and latin is dance oriented with high energy.

3. **Extracted the primary artist**: The `artists` column sometimes lists multiple artists separated by semicolons. I split on `;` and kept the first name as `primary_artist` for use as the join key.

4. **Cleaned the artists table**: Empty genre lists (`'[]'`) were replaced with `NaN`. Missing follower counts were filled with `0.0`. I deduplicated by keeping the entry with the highest follower count per artist name.

5. **Merged artist-level data**: Left join on `primary_artist` → `name`, adding `artist_followers` and `artist_popularity` to each track. Unmatched tracks received `artist_followers = 0.0` and `artist_popularity` = the column median.

6. **Created `is_popular`**: A binary label equal to `1` if popularity ≥ 70, else `0`. Used as the classification target in Steps 5–8.

The first few rows of the cleaned DataFrame (selected columns):

| track_name | track_genre | popularity | danceability | energy | acousticness | speechiness | artist_followers | is_popular |
|------------|-------------|-----------|-------------|--------|-------------|------------|-----------------|-----------|
| Just You and Me | country | 0 | 0.585 | 0.340 | 0.767 | 0.024 | 690,851 | 0 |
| Born Again | country | 0 | 0.476 | 0.888 | 0.013 | 0.053 | 906,755 | 0 |
| Go Amanda | country | 0 | 0.326 | 0.716 | 0.002 | 0.031 | 206,226 | 0 |
| Excitable Boy | country | 0 | 0.628 | 0.854 | 0.083 | 0.034 | 690,851 | 0 |
| Jerusalem | country | 0 | 0.445 | 0.776 | 0.003 | 0.035 | 206,226 | 0 |

### Univariate Analysis

The histogram below shows the distribution of track popularity scores broken down by genre.

<iframe
  src="assets/popularity_distribution.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Popularity scores are heavily skewed toward zero across all five genres, with a large spike at 0 representing tracks with little to no recent streaming activity. Hip-hop and latin tracks show a broader spread toward higher popularity values, while gospel and tango are more concentrated near zero suggesting that these genres attract smaller streaming audiences on Spotify.

### Bivariate Analysis

The scatter plot below shows the relationship between an artist's log₁₀ transformed follower count and their tracks' popularity scores, colored by genre.

<iframe
  src="assets/popularity_vs_followers.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Despite some clustering at the high follower, high popularity corner, the overall relationship between artist followers and track popularity is weak across all five genres. Tracks from artists with very few followers span the full popularity range, and many tracks from well followed artists score near zero, suggesting follower count alone is a poor predictor of streaming popularity.

### Interesting Aggregates

The table below shows mean audio features and mean popularity by genre, revealing each genre's distinct audio fingerprint.

| track_genre | danceability | energy | valence | acousticness | popularity |
|-------------|-------------|--------|---------|-------------|-----------|
| country     | 0.555 | 0.597 | 0.521 | 0.321 | 17.028 |
| gospel      | 0.473 | 0.576 | 0.321 | 0.377 | 41.639 |
| hip-hop     | 0.736 | 0.683 | 0.551 | 0.194 | 37.759 |
| latin       | 0.722 | 0.727 | 0.631 | 0.183 | 8.297  |
| tango       | 0.538 | 0.373 | 0.584 | 0.846 | 19.871 |

Tango stands out with very high acousticness and low energy, hip-hop leads in speechiness, latin tops danceability and valence, and gospel sits in a moderate energy, high-acousticness region. These distinct profiles directly motivate our genre classification task in Steps 5–8.

The pivot table below shows mean track popularity by genre and explicit content status.

| track_genre | Not Explicit | Explicit |
|-------------|-------------|---------|
| country     | 16.29 | 40.97 |
| gospel      | 41.64 | NaN   |
| hip-hop     | 44.54 | 23.29 |
| latin       | 7.89  | 10.95 |
| tango       | 19.87 | NaN   |

Gospel and tango have no explicit tracks in this dataset. Among genres that do have explicit content, the popularity gap between explicit and non explicit tracks varies by genre, hinting that explicit content may interact differently with streaming behavior depending on audience type.

---

## Assessment of Missingness

### NMAR Analysis

The column with the most meaningful missingness in our cleaned dataset is **`tempo`**, missing for approximately 21% of tracks. I believe `tempo` is **not NMAR** (Not Missing At Random). For tempo to be NMAR, its missingness probability would need to depend on the unobserved tempo value itself — for example, Spotify's beat-detection algorithm failing specifically on tracks with unusually high or low BPM. While mechanistically plausible, our permutation tests below show that missingness is well-explained by observed columns (`track_genre` and `energy`), which is consistent with MAR rather than NMAR.

To rule out NMAR more definitively, we would want: raw Spotify API response logs showing whether tempo extraction returned an error vs. a null, or a flag for tracks with irregular or free-form time signatures. If very fast or very slow tempos are disproportionately missing, that would support NMAR; if failure rates are uniform within genre and energy strata, that supports MAR.

### Missingness Dependency

I performed three permutation tests assessing whether `tempo` missingness depends on other columns.

**Test 1: `tempo` missingness vs. `track_genre` — dependent (MAR)**

Test statistic: Total Variation Distance (TVD) between the genre distribution of tracks with missing tempo and tracks without. After 1,000 permutations, the observed TVD was **0.1040** with a p-value of **0.0000**. Since p < 0.05, we reject the null and conclude tempo missingness **does depend on genre** — tango in particular contains many acoustic and arhythmic recordings where Spotify's beat-detection fails at higher rates.

<iframe
  src="assets/missingness_genre.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

*Empirical null distribution of TVD (genre). The red line marks the observed statistic.*

**Test 2: `tempo` missingness vs. `energy` — dependent (MAR)**

Test statistic: absolute difference in mean energy between missing and non-missing groups. The observed difference was **0.0971** with a p-value of **0.0000**. Since p < 0.05, we reject the null and conclude tempo missingness **does depend on energy** — low-energy tracks (quiet, sparse recordings) are less likely to have a detectable beat.

**Test 3: `tempo` missingness vs. `mode` — not dependent**

Test statistic: TVD between mode distributions (major vs. minor) of missing and non-missing groups. Observed TVD was **0.0150** with a p-value of **0.3770**. Since p ≥ 0.05, we fail to reject the null — tempo missingness **does not depend on musical mode**. Whether a track is in a major or minor key has no mechanistic connection to whether Spotify can detect its tempo.

---

## Hypothesis Testing

**Research question**: Do tracks from artists with more followers tend to be more popular?

**Null Hypothesis (H₀)**: The mean track popularity is the same for tracks from high-follower artists and tracks from low-follower artists. Any observed difference is due to random chance.

**Alternative Hypothesis (H₁)**: Tracks from artists with more followers have strictly higher mean track popularity.

**Groups**: Tracks are split at the median of `artist_followers`. Tracks above the median form the high-follower group; tracks at or below form the low-follower group.

**Test statistic**: Mean popularity (high-follower) − Mean popularity (low-follower). A positive value would support H₁. We use a one-sided test because our alternative hypothesis is directional.

**Significance level**: α = 0.05

**Method**: Permutation test with 10,000 iterations. We shuffle the popularity labels, breaking any real association between follower count and popularity, and compute the null distribution of the test statistic. No distributional assumptions are needed — appropriate because popularity is heavily right-skewed.

**Result**: The observed mean difference was **−3.2655**. The one-sided p-value was **1.0000**.

Since p ≥ 0.05, we **fail to reject H₀**. There is no statistically significant evidence that tracks from high-follower artists are more popular. The observed difference was actually negative, suggesting that within these five genres, a large following does not translate to higher track-level popularity. This is consistent with how Spotify's popularity metric works — it reflects *recency of streaming activity*, not artist prestige. A track from a lesser-known artist can score highly if it was recently streamed frequently, while an archived track from a major artist may score near zero.

<iframe
  src="assets/hypothesis_test.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

*Null distribution from 10,000 permutations. The red line marks the observed test statistic.*

---

## Framing a Prediction Problem

Our hypothesis test found that artist follower count is not a reliable predictor of track popularity. This pivots us toward a more tractable question driven by the EDA: **can we identify a track's genre purely from its audio features and metadata?**

**Prediction problem**: Predict the `track_genre` of a Spotify track from its audio features and metadata. This is a **5-class multiclass classification** problem (hip-hop, country, gospel, tango, latin).

**Response variable**: `track_genre`. Genre was chosen because the EDA aggregate table showed that each genre has a highly distinct audio fingerprint — making this a tractable and meaningful classification problem. Genre prediction is also practically useful, powering automatic music tagging, playlist generation, and content recommendation.

**Evaluation metric**: **Accuracy**. All five genres contribute exactly 1,000 tracks each to our filtered dataset, so the classes are perfectly balanced. Accuracy is therefore not misleading — a naïve random classifier would achieve exactly 20% (1/5 classes), giving us a clear performance floor. We prefer accuracy over macro-F1 because there is no meaningful cost asymmetry between misclassifying different genres.

**Features known at prediction time**: When a new track is uploaded to Spotify, the platform immediately computes all audio features (`danceability`, `energy`, `key`, `loudness`, `mode`, `speechiness`, `acousticness`, `instrumentalness`, `liveness`, `valence`, `tempo`, `time_signature`) and records `explicit` and `duration_ms`. All of these are available at the time of prediction.

I **excluded** the following:
- `popularity` — accumulates over time; unknown for a brand-new track
- `artist_followers` / `artist_popularity` — not intrinsic to the track and shown in Step 4 to be uninformative
- `release_year` — adds noise without a clear genre signal in this dataset

---

## Baseline Model

**Model**: `DecisionTreeClassifier(max_depth=3)` inside a single `sklearn` Pipeline.

**Features** (3 total):

| Feature | Type | Encoding |
|---------|------|---------|
| `energy` | Quantitative | Passthrough (no transformation) |
| `acousticness` | Quantitative | Passthrough (no transformation) |
| `explicit` | Nominal | `OneHotEncoder(drop='if_binary')` → single 0/1 column |

- **2 quantitative features**: `energy` and `acousticness`. These two were selected because the EDA aggregate table showed the largest cross-genre differences on precisely these dimensions — tango's acousticness (~0.85) vs. hip-hop's (~0.19) is the single most separating feature in the dataset.
- **0 ordinal features**
- **1 nominal feature**: `explicit`, encoded with `OneHotEncoder(drop='if_binary')` to produce a single binary column. Explicit content is strongly genre-correlated (absent from gospel and tango, common in hip-hop).

A shallow decision tree (max depth = 3, at most 8 leaf nodes) was chosen as the simplest interpretable classifier that can still capture non-linear genre boundaries, while capping depth at 3 prevents memorizing the training set.

**Performance**:

| Split | Accuracy |
|-------|---------|
| Train | 0.4670 |
| Test | 0.4480 |
| Random baseline | 0.2000 (5 balanced classes) |

The baseline achieves substantially better-than-random accuracy, confirming that `acousticness` and `energy` alone carry genuine genre signal. However, with only two quantitative features and very limited depth, the model cannot capture the full complexity of genre boundaries — motivating the improvements in Step 7.

---

## Final Model

**Two new engineered features**:

**1. `energy_x_dance`** = `energy` × `danceability`

This interaction term captures "dance energy" — the combination of perceptual intensity and rhythmic suitability for dancing. Hip-hop and latin score high on *both* dimensions simultaneously, while tango (low energy, moderate danceability) and gospel (low on both) occupy clearly different regions of this 2-D space. A product feature makes that joint relationship available to the classifier as a single dimension, which matters especially for tree-based models that can only split on one feature at a time.

**2. Binarized `speechiness`** (threshold = 0.33)

Per the Spotify dataset documentation, `speechiness > 0.33` indicates the track contains significant rap or vocal content. This threshold cleanly separates hip-hop (most tracks above 0.33) from all other genres. Binarizing at this semantically meaningful cut-point is more informative than treating speechiness as a continuous linear scale — the genre signal lies in whether the track *crosses* the threshold, not in the exact value above it.

Additionally, `SimpleImputer(strategy='median')` was applied to the `tempo` column (21% missing, MAR per Step 3), allowing the full feature to contribute without dropping rows. Median imputation is more robust than mean imputation given tempo's skewed distribution.

**Model**: `RandomForestClassifier` inside a single `sklearn` Pipeline, with hyperparameters selected via 5-fold `GridSearchCV`.

**Hyperparameter search**:

| Hyperparameter | Values Searched | Best Value |
|----------------|----------------|-----------|
| `n_estimators` | 100, 200 | 200 |
| `max_depth` | 10, 20, None | None |
| `min_samples_leaf` | 1, 3 | 1 |

`n_estimators` was tuned because more trees reduce ensemble variance. `max_depth` controls the bias-variance tradeoff — unlimited depth allows complex genre boundaries but risks overfitting. `min_samples_leaf` acts as regularization, preventing leaves that memorize tiny clusters.

**Performance**:

| Model | Train Accuracy | Test Accuracy |
|-------|---------------|--------------|
| Baseline (Decision Tree, depth=3) | 0.4670 | 0.4480 |
| Final (Random Forest + GridSearchCV) | 0.9805 | 0.7790 |
| Improvement | +0.5135 | +0.3310 |

The Final Model improves substantially over the baseline. The Random Forest benefits from the full audio feature suite, the `energy_x_dance` interaction term, and the semantically grounded `speechiness` binarization — all of which expose genre-discriminative signals that the shallow decision tree could not capture.

<iframe
  src="assets/confusion_matrix.html"
  width="600"
  height="550"
  frameborder="0"
></iframe>

*Confusion matrix on the held-out test set. Rows are actual genres; columns are predicted genres.*

---

## Fairness Analysis

**Group X**: Explicit tracks (`explicit = True`)
**Group Y**: Non-explicit tracks (`explicit = False`)

**Rationale**: In our five-genre dataset, explicit content appears almost exclusively in hip-hop, country, and latin tracks — gospel and tango have zero explicit tracks. This means the model may have implicitly learned that `explicit = True` is a strong genre signal, potentially creating a systematic accuracy gap between the two groups.

**Evaluation metric**: Accuracy (same as the overall evaluation metric).

**Null Hypothesis (H₀)**: The model's accuracy is the same for explicit and non-explicit tracks. Any observed difference is due to random chance in group membership.

**Alternative Hypothesis (H₁)**: The model's accuracy differs between explicit and non-explicit tracks.

**Test statistic**: |accuracy(explicit) − accuracy(non-explicit)| — two-sided, since we have no prior reason to expect which direction any bias would run.

**Significance level**: α = 0.05

**Method**: Permutation test with 1,000 iterations. We fix the model's predictions on the test set and shuffle only the explicit/non-explicit group labels. The model is never refit during the test.

**Results**:

| Group | Accuracy |
|-------|---------|
| Explicit tracks | 0.7798 |
| Non-explicit tracks | 0.7789 |
| Observed \|difference\| | 0.0009 |
| p-value (two-sided) | 1.0000 |

Since p ≥ 0.05, we **fail to reject H₀**. There is no statistically significant evidence of an accuracy gap between explicit and non-explicit tracks. Any observed difference is consistent with random variation in group membership. The model appears to be **fair with respect to explicit content status**.

<iframe
  src="assets/fairness_test.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

*Null distribution of |accuracy difference| from 1,000 permutations. The red line marks the observed statistic.*
