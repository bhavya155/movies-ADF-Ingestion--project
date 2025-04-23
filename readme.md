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

SELECT year(from_unixtime(1377476993)) AS rating_year, COUNT(*) AS total_ratings 
FROM db_projects.dev.ratings_sv 
GROUP BY rating 
ORDER BY rating;

3.2 Rating Distribution

df=spark.sql("SELECT rating, COUNT(*) AS count
             FROM db_projects.dev.ratings_sv
             GROUP BY rating 
             ORDER BY rating")
df.display()

3.3 18 Movies Tagged but Not Rated

SELECT DISTINCT t.movie_id,t.tag 
FROM db_projects.dev.tags_sv t 
LEFT JOIN db_projects.dev.ratings_sv r 
ON t.movie_id = r.movie_id 
WHERE r.movie_id IS NULL 
LIMIT 18;

3.4 Top 10 Untagged Rated Movies (30+ Ratings)

SELECT movie_id, AVG(rating) AS avg_rating, COUNT(*) AS num_ratings 
FROM db_projects.dev.ratings_sv 
WHERE movie_id 
NOT IN (SELECT DISTINCT movie_id FROM db_projects.dev.tags_sv) 
GROUP BY movie_id 
HAVING COUNT(*) > 30 
ORDER BY avg_rating DESC, num_ratings DESC 
LIMIT 10;

✅ Outcome

- Scalable and traceable data ingestion pipeline
- Clean, query-optimized Delta Lake storage (Silver Layer)
- Ready for BI dashboards or ML models
