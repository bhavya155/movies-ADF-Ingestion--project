Movies-Ratings Azure Data Engineering Document
Architecture Diagram

This diagram shows the complete data flow:
![movies_ratings flow diagram](https://github.com/user-attachments/assets/660f8452-aed1-4bbf-b207-c76895e74971)

 

1. Sources:
   - HTTP endpoints
   - On-Premises SQL Server
   - Azure Blob Storage

2. Ingestion Tool: Azure Data Factory orchestrates the ingestion from all sources into Azure Data Lake Storage.

3. Storage Layer: Raw data is stored in hierarchical folders inside Azure Data Lake.

4. Transformation: Azure Databricks processes raw data, cleans and enriches it, and stores it in the Silver Layer for analytics.

1. Ingestion Layer
1.1 Setup

- Blob Storage Account created with landing_ratings container.
- Files involved:
  - ratings.csv from Blob Storage
  - movies.csv from GitHub
  - tags.csv uploaded to On-Prem SQL Server (tags table)

1.2 Ingestion Flow

- Ingest ratings.csv to movies/raw_ratings in ADLS
- Ingest movies.csv to movies/raw_movies in ADLS
- Ingest tags table (from on-prem SQL Server) to movies/raw_tags in ADLS

Tools Used: Azure Data Factory with:
- Copy activity for HTTP and Blob
- Self-hosted Integration Runtime for On-Premises SQL

2. Transformation Layer (Azure Databricks)
2.1 Cleaning & Enrichment

- Rename Columns to standardize schema for Delta Lake
- Add Metadata:
  - ingestion_date (current date)
  - source_file_name (for traceability)
- Validation:
  - Drop duplicates
  - Drop unwanted columns
  - Schema enforcement and alignment

2.2 Save to Silver Layer

- Format: Delta Lake
- Location: movies/silver in ADLS
- Accessible to BI & Data Science teams for analytics

3. Analysis Layer (Querying the Silver Tables)
3.1 Ratings per Year

SELECT YEAR(to_date(timestamp_column)) AS rating_year, COUNT(*) AS total_ratings
FROM silver_ratings
GROUP BY rating_year
ORDER BY rating_year;

3.2 Rating Distribution

SELECT rating, COUNT(*) AS count
FROM silver_ratings
GROUP BY rating
ORDER BY rating;

3.3 18 Movies Tagged but Not Rated

SELECT DISTINCT t.movieId
FROM silver_tags t
LEFT JOIN silver_ratings r ON t.movieId = r.movieId
WHERE r.movieId IS NULL
LIMIT 18;

3.4 Top 10 Untagged Rated Movies (30+ Ratings)

SELECT movieId, AVG(rating) AS avg_rating, COUNT(*) AS num_ratings
FROM silver_ratings
WHERE movieId NOT IN (SELECT DISTINCT movieId FROM silver_tags)
GROUP BY movieId
HAVING COUNT(*) > 30
ORDER BY avg_rating DESC, num_ratings DESC
LIMIT 10;

3.5 Tagging Statistics - Average Tags per Movie

SELECT COUNT(*) * 1.0 / COUNT(DISTINCT movieId) AS avg_tags_per_movie
FROM silver_tags;

3.5 Tagging Statistics - Average Tags per User

SELECT COUNT(*) * 1.0 / COUNT(DISTINCT userId) AS avg_tags_per_user
FROM silver_tags;

3.5 Tagging Statistics - Avg Tags per User per Movie

SELECT AVG(tag_count) AS avg_tags_per_user_movie
FROM (
  SELECT userId, movieId, COUNT(*) AS tag_count
  FROM silver_tags
  GROUP BY userId, movieId
);

3.6 Users Who Tagged but Didn’t Rate

SELECT DISTINCT t.userId
FROM silver_tags t
LEFT JOIN silver_ratings r ON t.userId = r.userId AND t.movieId = r.movieId
WHERE r.rating IS NULL;

✅ Outcome

- Scalable and traceable data ingestion pipeline
- Clean, query-optimized Delta Lake storage (Silver Layer)
- Ready for BI dashboards or ML models
