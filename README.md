# Faculty Profile Analysis: IISER Mohali

**Topic 4: Web Scraping II, Faculty Profiles (Course project for IDC409)**

Authors: Shashvita Ratra (MS24190), Stavya Pandey (MS24245)

Instructor: Dr. Vishal Bharadwaj

Semester: Monsoon Semester 2026-27

This project scrapes the faculty profile pages of IISER Mohali, cleans and stores them in a SQLite database, explores the data and then applies text-mining and machine-learning methods to the faculty's research descriptions. The aim is to find patterns in training (PhD year and institution, postdoc and career path) and expertise (research areas and interests).

---

## Methodology

| Stage | Notebook | Summary |
| :---- | :---- | :---- |
| 1\. Scrape | `scrapper.ipynb` | Checks `robots.txt`, scrapes the People page listing, then visits each faculty profile and collects Research Area, Research Interests, Career Highlights, Awards and Publications. |
| 2\. Clean | `Cleaning.ipynb` | Removes duplicates, parses free-text career entries into PhD year, PhD institution, postdoc and professor-level positions, adds publication/award flags, and writes the raw and clean tables to SQLite. |
| 3\. Explore | `EDA.ipynb` | Missing values, duplicates, department/designation/category counts, PhD year and institution trends, profile completeness, text lengths. Writes the final table back to the database. |
| 4\. Model | `Clustering.ipynb` | TF-IDF keyword extraction, K-Means clustering of research areas, similarity search, department classification, PhD-origin analysis. |

Run the notebooks in the order above. Each one reads the output of the previous one.

---

## Data

- **Source:** the IISER Mohali People page and the individual faculty profile pages linked from it.  
- **Scope:** one institution, to keep the scope manageable.  
- **Size:** 135 records after removing 1 duplicate (136 raw), with 20 columns in the final table.  
- **Composition:** 122 Regular Faculty, 5 Honorary Professors, 3 Visiting Faculty, 2 Lecturers, 2 Emeritus Professors, 1 Adjunct Faculty.  
- **Departments in the modelling set:** DCS, DBS, DPS, DMS, EES, HSS.  
- **Scraping notes:** 127 profile pages were on the institute's own site and were visited. One page (`tv-venkateswaran`) returned HTTP 403 and could not be scraped.

### Database

`iisermohali_faculty.db` (SQLite) holds these tables:

| Table | Written by | Contents |
| :---- | :---- | :---- |
| `faculty_raw` | `Cleaning.ipynb` | Scraped data as collected |
| `faculty_clean` | `Cleaning.ipynb`, updated by `EDA.ipynb` | Cleaned data with parsed career fields and EDA features |
| `faculty_clusters` | `Clustering.ipynb` | Keywords, cluster id, predicted department and t-SNE coordinates |
| `cluster_terms` | `Clustering.ipynb` | Top terms for each cluster |

---

## Methods

**Scraping.** `requests` with a retry/backoff session, `BeautifulSoup` for parsing. Before scraping, the script reads `robots.txt` and checks that the People page is allowed. Faculty categories that are not of interest (administrative staff, technical staff, former faculty, institute fellows, project scientists) are skipped.

**Cleaning.** Career Highlights are split into individual dated entries with regular expressions. Each entry is then classified as PhD, Postdoc, Assistant/Associate Professor, Professor or Other. PhD year and institution are extracted per person.

**Text processing.** Research Area and Research Interests are combined (Research Area is weighted double). Trailing biography, publication lists and degree details are trimmed. Text is lowercased, URLs and emails are stripped, and generic words ("research", "group", "study" and so on) are removed on top of the standard English stop words.

**Features.** TF-IDF with unigrams and bigrams (`min_df=2`, `max_df=0.5`, sublinear TF) gives a 110 × 1041 matrix (97% sparse). For clustering it is reduced to 20 dimensions with Truncated SVD and L2-normalised.

**Models.**

- K-Means clustering (k \= 10\) on the reduced features. Silhouette score, seed stability (ARI) and agreement with departments were compared across k \= 6 to 14 before choosing k.  
- Logistic Regression and Linear SVM predicting department from profile text, evaluated with stratified 5-fold cross-validation against a majority-class baseline.  
- Cosine-similarity search to find faculty with similar expertise, and free-text query search.

---

## Results

**Clustering**

- 110 regular faculty with usable text were clustered into 10 groups of 7 to 15 people each.  
- Silhouette score (cosine) is 0.296.  
- Adjusted Rand Index between clusters and departments is 0.507, so research-text clusters partly follow department lines but cut across them as well.

**Department classification (5-fold CV)**

| Model | Accuracy | Macro F1 |
| :---- | :---- | :---- |
| Majority-class baseline | 0.218 | n/a |
| Logistic Regression | 0.864 | 0.843 |
| **Linear SVM** | **0.873** | **0.859** |

- 14 of 110 faculty were misclassified.  
- EES has the lowest recall (0.58). Many misclassifications are interdisciplinary profiles, such as a machine-learning researcher in DCS predicted as DPS.

**PhD origin.** PhD institution was available for 96 of 122 regular faculty. The largest groups are other Indian institutions (CSIR labs, JNU, DU, BHU and similar), then TIFR/IMSc/HRI/ISI/NCBS, then IITs, then European universities.

**Similarity search.** Given a faculty name or a free-text query (for example "quantum condensed matter"), the system returns the most similar profiles with a similarity score.

---

## EDA: key observations

(Computed from `faculty_eda_done.csv`; 135 people, 122 regular faculty.)

- **Department sizes.** Across all 135 records: DPS 32, DBS 31, DCS 26, DMS 23, EES 13, HSS 10. Among the 122 regular faculty: DPS 28, DBS 28, DCS 25, DMS 21, EES 12, HSS 8.
- **Seniority is balanced.** Regular faculty: 44 Assistant Professors, 40 Associate Professors, 38 Professors.
- **PhD cohort.** Of the 96 regular faculty with a PhD year, 53 (55%) started their PhD in the 2000s, 22 in the 1990s, 18 in the 2010s, 2 in the 1980s and 1 in the 2020s.
- **Postdoc entries vary strongly by department.** A postdoc entry was parsed for about 82% of DBS regular faculty, 52% of DCS, 50% of EES, 32% of DPS, 25% of HSS and 24% of DMS. This reflects what profiles list as well as real career paths.
- **Missing data.** Of 122 regular faculty, PhD year/institution is missing for 26 (21%) and Awards for 43 (35%); 89% have a Publications section.
- **Profile completeness (maximum 8).** Median 8, mean 7.0. DMS profiles are the least complete (mean 6.3), EES the most (7.8).
- **Text fields are not clean.** The scraped "Research Interests" field contains the whole tab (median about 47,600 characters, maximum about 785,000), so it had to be trimmed before text analysis.

---

## Limitations

- **Small sample.** 110 profiles are used for modelling, and the smallest department has only 8\. Cross-validation scores will move noticeably with a different split.  
- **Moderate cluster quality.** A silhouette of 0.296 means the clusters overlap. The 20 SVD components keep only 34.1% of the variance, since short, sparse text is hard to compress.  
- **Missing data.** PhD institution is missing for 26 of 122 regular faculty. One profile could not be scraped (HTTP 403).  
- **Rule-based parsing.** Career entries and PhD-institution buckets are extracted with regular expressions and keyword rules. Unusual formats can be mislabelled, and 8 institutions fell into "Unclassified".  
- **Single institution.** Comparisons with other institutes (IIT, NIT, other IISERs) are not covered.  
- **Labels and text overlap.** Department labels are partly encoded in the research text itself, so classification accuracy overstates how well expertise is predicted from scratch.  
- **Residual noise in parsed fields.** One PhD year range is reversed (2024-2018), some PhD-institution strings are untidy, 11 names keep "Prof."/"Dr." prefixes, and most Email entries hold a username without the domain.  
- **Postdoc and award fields reflect what profiles list**, not verified career histories.

Possible extensions: add other institutes' profiles, map research keywords across institutes, and try embedding-based representations instead of TF-IDF.

---

## Course requirements

| \# | Requirement | Where it is met |
| :---- | :---- | :---- |
| 1 | Use numpy and pandas to store data | All notebooks |
| 2 | Database: read, write and modify | SQLite tables created, queried, inserted into, updated and deleted (on a scratch copy) in `Database_Demo.ipynb`; written in `Cleaning.ipynb`, `EDA.ipynb` and `Clustering.ipynb` |
| 3 | Exploratory Data Analysis | `EDA.ipynb` |
| 4 | Understand features, and reduce them | TF-IDF feature extraction, `min_df`/`max_df` filtering, Truncated SVD |
| 5 | Machine-learning models | K-Means, Logistic Regression, Linear SVM |
| 6 | Evaluate with metrics | Silhouette, ARI, accuracy, macro F1, confusion matrix |
| 7 | Share code on GitHub | This repository |

---

## How to run

**Requirements:** Python 3.9+ and the packages below.

```sh
pip install pandas numpy matplotlib seaborn scikit-learn beautifulsoup4 requests wordcloud jupyter
```

**Order of execution**

1. `scrapper.ipynb` writes `iisermohali_faculty_listing.csv` and `iisermohali_faculty_raw_final.csv`.  
2. `Cleaning.ipynb` reads a copy of the raw CSV and writes `faculty_career_summary.csv`, `summary_of_faculty.csv`, `iisermohali_faculty_clean.csv` and the database.  
3. `EDA.ipynb` reads a copy of the clean CSV and writes `faculty_eda_done.csv`, then updates the database.  
4. `Clustering.ipynb` reads `faculty_eda_done.csv` and writes `faculty_clusters.csv` plus the `faculty_clusters` and `cluster_terms` tables.

**File paths.** `Clustering.ipynb` expects the files in `/content/` (Google Colab), set by `DATA_PATH` and `DB_PATH` in the second code cell. The other notebooks use relative paths. `Cleaning.ipynb` and `EDA.ipynb` read copies of earlier outputs that were renamed by hand (`..._Copy.csv`, `iisermohali_faculty_complete_raw.csv`), so make sure those copies exist or edit the filenames before running.

---

## Repository structure

```
.
├── README.md
├── scrapper.ipynb       # scraping
├── Cleaning.ipynb       # cleaning + SQLite
├── EDA.ipynb            # exploratory analysis
├── Clustering.ipynb     # TF-IDF, clustering, search, classification
├── Database_Demo.ipynb  # SQL read, insert, update, delete demonstration
├── iisermohali_faculty.db
└── *.csv                # intermediate and final data files
```

---

## Ethical note

The scraper checks `robots.txt` (the People page and every profile page), identifies itself with the User-Agent `IISERFacultyResearch/1.0`, waits at least 1 second between requests, and retries failed requests with backoff. The data in this repository was collected in an earlier run, before these safeguards were complete: that run did not pause between requests and, because of a header-name typo, did not send the custom User-Agent. The code has since been corrected. A verification re-run on November 4, 2026 was not blocked by robots.txt and returned the same 135 rows and 11 columns as the original; one row (Shravan Kumar Mishra) differed in four columns: Research Interests, Career Highlights, Awards, and Publications (reflecting live updates made to the faculty member's profile page on the institute website). The original data was kept. The data consists of publicly listed faculty profile information and is used only for academic analysis. Please respect the institute's terms of use if you re-run it.
