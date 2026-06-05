# Happy Songs, Hit Songs? Emotion, Audio, and Popularity in Spotify Tracks

By **Kitty Ming** and **Trang Kieu**.

---

## Introduction

Streaming platforms and artists constantly ask the same question: what makes a
song resonate with listeners? One intuitive guess is *emotional tone* — maybe
upbeat, cheerful songs are simply more popular than sad ones. We put that idea to
the test.

This project investigates a single question:

> **Are happier songs more popular than sadder songs across different Spotify genres?**

To explore it, we use Spotify's **valence** feature, which measures the musical
positiveness of a track on a 0–1 scale: higher valence means a happier, more
cheerful song, while lower valence means a sadder, more negative one. We compare
valence against Spotify's **popularity** score to ask whether the emotional
character of music is associated with how popular it becomes.

**The data.** The full dataset (`music_tracks.csv`) contains 114,000 tracks
spanning 114 genres, with one row per (track, genre) pairing. Each row records
identifying metadata (track name, artist, album, release date) alongside a set of
numeric audio features that Spotify computes automatically from the audio signal.
To keep the analysis focused and the genres musically distinct, we restrict
attention to five genres — **pop, rock, hip-hop, classical, and edm** — with 1,000
tracks each, for 5,000 rows total. These five differ substantially in rhythm,
instrumentation, production style, and emotional tone, which makes them ideal for
comparing feature distributions.

The columns most relevant to our question are:

| Column | Description |
|---|---|
| `valence` | Musical positiveness, 0.0 (sad) to 1.0 (happy). Our measure of "happiness." |
| `popularity` | Spotify popularity score, 0–100. The outcome we care about. |
| `track_genre` | The genre label for the track. Used to compare across genres. |
| `energy` | Perceptual intensity / activity, 0.0–1.0. |
| `acousticness` | Confidence the track is acoustic, 0.0–1.0. |
| `danceability`, `tempo`, `loudness`, `speechiness`, `instrumentalness`, `liveness` | Additional audio features used for modeling and missingness analysis. |
| `duration_ms`, `explicit`, `release_date` | Track metadata. |


Why should we care? If emotional tone really did predict popularity, that would
be a simple, interpretable lever for anyone making or curating music. As we'll
see, the real story turns out to be more nuanced.

---

## Data Cleaning and Exploratory Data Analysis

### Data cleaning

We performed the following cleaning steps, each motivated by how the data was
generated:

1. **Loaded both files.** `music_tracks.csv` holds the track-level audio features;
   `artists.csv` holds artist-level metadata, which we keep available even though
   our question is track-level.
2. **Subset to relevant columns and to our five genres.** This drops the 109
   genres we are not studying and keeps only the columns relevant to the question
   and to later modeling.
3. **Parsed `release_date` into `release_year`.** The raw field is stored
   inconsistently — some rows are `"1974"`, others `"1995-04"` or
   `"2018-08-10"`. We take the first four characters as the year and coerce to
   numeric, turning anything unparseable into `NaN`.
4. **Converted `duration_ms` to `duration_min`** for interpretability.
5. **Created a `happiness_group` label** by thresholding valence at 0.5
   (`Happier` if `valence >= 0.5`, else `Sadder`). This binarization directly
   operationalizes our research question.

**A note on `tempo`.** About 20% of `tempo` values are missing. We deliberately do
**not** drop or impute these rows here, because the missingness of `tempo` is the
subject of the next section, so we keep the `NaN`s intact.

**A note on duplicates.** Because the dataset has one row per (track, genre), a
song that Spotify tags with multiple genres can appear more than once. For a
genre-level comparison this is acceptable: we treat each (track, genre) pairing as
the unit of observation.

The head of our cleaned DataFrame (a selection of the most relevant columns):

| track_name             | track_genre   |   popularity |   release_year |   duration_min |   valence |   energy | happiness_group   |
|:-----------------------|:--------------|-------------:|---------------:|---------------:|----------:|---------:|:------------------|
| Zara Zara              | classical     |           58 |           2001 |          4.971 |     0.62  |    0.268 | Happier           |
| Kajra Re               | classical     |           59 |           2005 |          8.043 |     0.68  |    0.898 | Happier           |
| Zara Zara - Lofi       | classical     |           54 |           1984 |          3.657 |     0.439 |    0.638 | Sadder            |
| Vaseegara              | classical     |           68 |           1972 |          4.986 |     0.637 |    0.293 | Happier           |
| Zara Zara - LoFi Chill | classical     |           59 |           1987 |          6.462 |     0.241 |    0.308 | Sadder            |

### Univariate analysis

<iframe src="assets/univariate-valence.html" width="100%" height="500" frameborder="0"></iframe>

The distribution of valence is fairly spread out with a slight concentration near
the middle of the scale — very few songs are extremely sad or extremely happy.

<iframe src="assets/popularity-distribution.html" width="100%" height="500" frameborder="0"></iframe>

Popularity is strongly right-skewed: most tracks have low-to-moderate popularity,
and only a small number of tracks are highly popular.

### Bivariate analysis

<iframe src="assets/popularity-by-happiness.html" width="100%" height="500" frameborder="0"></iframe>

When we split tracks into "Happier" and "Sadder" groups, the two popularity
distributions look almost identical. This is an early hint that valence alone may
not separate popular songs from unpopular ones.

<iframe src="assets/valence-by-genre.html" width="100%" height="500" frameborder="0"></iframe>

Valence, however, varies a lot *across genres*: EDM and pop skew happier (higher
valence), while classical skews much sadder (lower valence). Genres clearly differ
in emotional tone even if happiness doesn't track popularity.

### Interesting aggregates

Grouping by genre and averaging several audio features summarizes how the five
genres differ. Note how classical has by far the lowest valence and energy and the
highest acousticness, while EDM and hip-hop are high-energy:

| track_genre   |   valence |   popularity |   energy |   acousticness |
|:--------------|----------:|-------------:|---------:|---------------:|
| classical     |     0.381 |       13.055 |    0.19  |          0.92  |
| edm           |     0.465 |       35.032 |    0.756 |          0.114 |
| hip-hop       |     0.551 |       37.759 |    0.683 |          0.194 |
| pop           |     0.506 |       47.576 |    0.606 |          0.344 |
| rock          |     0.539 |       19.001 |    0.679 |          0.209 |

This table is significant because it shows the genres are genuinely distinct
audio "profiles." Pop is the most popular on average despite only middling valence,
and classical, the saddest and most acoustic genre, is the least popular. That
mismatch between valence and popularity foreshadows our hypothesis-test result.

---

## Assessment of Missingness

### NMAR analysis

We do **not** believe any column in our dataset is **NMAR** (Not Missing At
Random). The only column with substantial missingness is `tempo`. Spotify derives
tempo from an automated beat-tracking algorithm, so whether a value gets recorded
depends on properties of the audio itself: beat tracking tends to fail on music
with a weak or ambiguous pulse (ambient, classical, solo piano) and succeeds
easily on music with a strong, regular beat (EDM, metal). This makes the
missingness of `tempo` most likely **MAR**, explained by other observed audio
features such as energy and acousticness, rather than NMAR, which would require
the missingness to depend on the hidden tempo value itself.

We cannot fully rule out NMAR without additional data from Spotify's pipeline,
such as a beat-tracking confidence flag recording exactly *why* tempo failed to
compute for a given track. If such a flag fully explained the missingness, we
could be confident the mechanism is MAR. We test the MAR claim empirically below.

### Missingness dependency

We defined an indicator `tempo_missing` (about **20.5%** of rows) and asked whether
the distribution of another column differs between rows where `tempo` is missing
versus present. For each candidate column we used the **difference in group means**
as the test statistic and ran a permutation test (shuffling the missingness labels
5,000 times) to obtain a p-value.

**Depends on `energy` (expected yes).** The observed difference in mean energy
between missing and non-missing tracks is about **−0.15**, meaning tracks with a
missing tempo tend to be much lower energy. This falls far outside the permutation
null distribution, giving **p ≈ 0**, so we reject the null: the missingness of
`tempo` *does* depend on energy.

<iframe src="assets/permtest-energy.html" width="100%" height="500" frameborder="0"></iframe>

**Does not depend on `duration_ms` (expected no).** The observed difference in
mean duration is well inside the null distribution, with **p ≈ 0.50**. We fail to
reject the null: there is no evidence that whether `tempo` is missing depends on how
long a track is.

Together these results support our reasoning that `tempo` is **MAR**. Its
missingness is explained by observed audio features like energy, not by the hidden
tempo value itself, and not by unrelated metadata like duration.

---

## Hypothesis Testing

We test our central question directly: is a song's happiness (valence) associated
with its popularity?

- **Null hypothesis (H₀):** There is no association between song valence and
  Spotify popularity across the selected genres. The true correlation is 0, and any
  observed correlation is due to chance.
- **Alternative hypothesis (H₁):** Songs with higher valence tend to have higher
  popularity (a positive association).
- **Test statistic:** the **Pearson correlation coefficient** r between `valence`
  and `popularity`. This is a natural choice because both variables are continuous
  and we are asking about a monotonic association between them; r directly measures
  the strength and direction of that linear relationship.
- **Significance level:** α = 0.05.
- **Method:** a permutation test. Under H₀, valence and popularity are unrelated,
  so we approximate the null distribution of r by repeatedly shuffling the
  `popularity` column and recomputing the correlation.

The observed correlation is **r ≈ 0.009**, essentially zero, with a two-sided
**p-value ≈ 0.52**.

<iframe src="assets/permtest-valence-pop.html" width="100%" height="500" frameborder="0"></iframe>

Since p ≈ 0.52 is far larger than 0.05, we **fail to reject the null hypothesis**.
We have no evidence that valence and popularity are related. (Failing to reject the
null is not the same as proving the two are unrelated. It only means we found no
convincing evidence of a relationship.) Popularity is likely driven much more by
factors outside the audio signal, such as artist fame, playlist placement, and
marketing, than by a song's emotional tone. This finding directly motivates our
prediction problem: rather than expecting valence to drive popularity, we ask how
well the full set of audio features can predict it.

---

## Framing a Prediction Problem

The hypothesis test showed that no single audio feature (valence) explains
popularity. A natural follow-up is to ask whether *all* the audio features together
carry any predictive signal:

> **Can we predict a track's `popularity` score from its audio features?**

- **Problem type:** **regression**, since the response is a continuous score from 0 to 100.
- **Response variable:** `popularity`. We chose it because it is the outcome at the
  heart of our original question ("what makes a song popular?"), and predicting it
  lets us quantify exactly how much of popularity the audio signal can and cannot explain.
- **Features available at "time of prediction":** only intrinsic audio
  characteristics of the track — `danceability`, `energy`, `acousticness`,
  `speechiness`, `instrumentalness`, `valence`, `loudness`, `liveness` (plus
  `track_genre`, `explicit`, duration, and release year). All of these are known
  before a track is released and before any popularity is observed. We deliberately
  exclude anything that is a *consequence* of popularity (e.g., follower counts) and
  any free-text identifier fields.
- **Evaluation metric:** **RMSE (root mean squared error)**. RMSE is the standard,
interpretable error measure for regression, reporting the typical prediction
error in the same units as popularity (points on the 0 to 100 scale). We prefer it
over R² as the headline metric because RMSE is directly comparable between the
baseline and final models and is easy to interpret. We also report R² as a
secondary measure of variance explained.

---

## Baseline Model

Our baseline model is a **linear regression** trained inside a single
`sklearn` Pipeline on two quantitative features: **valence** and **energy**. Both
are continuous numeric columns that require no encoding, so the baseline uses **2
quantitative features, 0 ordinal, and 0 nominal**.

On the held-out 25% test set, the baseline achieves an **RMSE of 32.38** points and
an **R² of only 0.04**. For reference, naively predicting the mean popularity for
every track gives an RMSE of **33.10**, so the baseline barely improves on simply
guessing the mean.

Is this a good model? **No**, but that is an expected and informative result. It is
consistent with what we found in the hypothesis test: audio features have very
little *linear* relationship with popularity, so a two-feature linear model cannot
explain much variance. It is still a legitimate baseline to improve upon, which we
do next by adding more features and switching to a more flexible model.

---

## Final Model

For the final model we used a **Random Forest Regressor**, because the relationship
between Spotify audio features and popularity is likely nonlinear and full of
interactions that a linear model cannot capture. We kept the **same train/test
split** as the baseline so the comparison is fair.

**Features added.** On top of the eight audio features, we added categorical
encodings for `track_genre` and `explicit` (one-hot encoded), and engineered **four
new features**, each motivated by the data generating process:

- `energy_minus_acousticness` — high-energy / low-acoustic songs resemble
  dance/EDM/pop production, so this contrast helps separate production styles.
- `log_duration_min` — duration is right-skewed, so logging it reduces the influence
  of unusually long tracks.
- `release_age` ( = 2026 − release year) — Spotify popularity is *current*, so older
  music can systematically score lower; age captures this directly.
- `speech_to_danceability` — speechy tracks (rap, spoken-word) behave differently
  from melodic ones, and this ratio encodes that distinction.

These are good features because they encode *domain knowledge* about how popularity
and production interact, rather than blindly transforming columns. For example,
`release_age` captures the real-world fact that the popularity score reflects recent
listening.

**Hyperparameter selection.** We tuned four Random Forest hyperparameters with
5-fold `GridSearchCV` (scoring on negative RMSE): the number of trees
(`n_estimators`), the maximum tree depth (`max_depth`), the minimum samples per leaf
(`min_samples_leaf`), and the number of features considered at each split
(`max_features`). The best-performing combination was **`n_estimators = 200`,
`max_depth = None`, `min_samples_leaf = 1`, `max_features = 'sqrt'`**.

**Performance.** On the same held-out test set, the final model lowers RMSE from
**32.38 → 23.50** and raises R² from **0.04 → 0.50**. This is a clear improvement:
the model now explains roughly half the variance in popularity, confirming that
while no *single* feature predicts popularity, the audio features *together*
(especially in a nonlinear model with genre and recency) do carry real signal.

<iframe src="assets/final-model-rmse.html" width="100%" height="500" frameborder="0"></iframe>

---

## Fairness Analysis

Finally, we ask whether the final model performs equally well across genres. Does
it predict popularity worse for one kind of music than another?

- **Group X:** classical songs.
- **Group Y:** non-classical songs (pop, rock, hip-hop, edm).
- **Evaluation metric:** RMSE (this is a regression model, so classification metrics
  like precision do not apply).
- **Null hypothesis:** The model is fair. Its RMSE for classical and non-classical
  songs is roughly the same, and any observed difference is due to random chance.
- **Alternative hypothesis:** The model performs *worse* for classical songs — its
  RMSE for classical songs is higher than for non-classical songs.
- **Test statistic:** RMSE(Classical) − RMSE(Non-classical).
- **Significance level:** α = 0.05.

The permutation test holds the fitted final model fixed and shuffles only the group
labels 5,000 times. The observed RMSE for classical songs (**14.45**) is actually
*lower* than for non-classical songs (**25.40**), giving an observed difference of
about **−10.94** — the opposite direction of the alternative. The resulting
**p-value ≈ 1.0**, so we **fail to reject the null hypothesis**.

<iframe src="assets/fairness-permutation.html" width="100%" height="500" frameborder="0"></iframe>

In other words, we find no evidence that the model is unfair against classical
songs. If anything, it predicts classical popularity *more* accurately, ikely
because classical tracks cluster at the low end of the popularity scale, making them
easier to predict.

---

*Built with Jekyll and GitHub Pages for DSC 80 at UC San Diego. All interactive
plots were created with Plotly.*