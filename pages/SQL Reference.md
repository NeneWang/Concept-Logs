- Documentation:
	- Data Loading (STAGE, ALTER STAGE): https://docs.snowflake.com/en/sql-reference/commands-data-loading
- Getting Data Process
	- Load company metadata `.csv` file.
		- Create the Formatting
			- ```
			  CREATE OR REPLACE FILE FORMAT csv
			      TYPE = 'CSV'
			      COMPRESSION = 'AUTO'  -- Automatically determines the compression of files
			      FIELD_DELIMITER = ','  -- Specifies comma as the field delimiter
			      RECORD_DELIMITER = '\n'  -- Specifies newline as the record delimiter
			      SKIP_HEADER = 0  -- No headers to skip, starts reading from the first line
			      FIELD_OPTIONALLY_ENCLOSED_BY = '\042'  -- Fields are optionally enclosed by double quotes (ASCII code 34)
			      TRIM_SPACE = FALSE  -- Spaces are not trimmed from fields
			      ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE  -- Does not raise an error if the number of fields in the data file varies
			      ESCAPE = 'NONE'  -- No escape character for special character escaping
			      ESCAPE_UNENCLOSED_FIELD = '\134'  -- Backslash is the escape character for unenclosed fields
			      DATE_FORMAT = 'AUTO'  -- Automatically detects the date format
			      TIMESTAMP_FORMAT = 'AUTO'  -- Automatically detects the timestamp format
			      NULL_IF = ('')  -- Treats empty strings as NULL values
			      COMMENT = 'File format for ingesting data for zero to snowflake';
			  ```
		- Verify that the file format was correctly done:
			- ```
			  SHOW FILE FORMATS IN DATABASE cybersyn;
			  ```
		- Download the S3 File
			- **Stage Name**: `cybersyn_company_metadata`**URL**: `s3://sfquickstarts/zero_to_snowflake/cybersyn-consumer-company-metadata-csv/cybersyn_consumer_company_metadata.csv`
			-
		- Load the staging data
			- ```
			  COPY INTO company_metadata FROM @cybersyn_company_metadata file_format=csv PATTERN = '.*csv.*' ON_ERROR = 'CONTINUE';
			  ```
		- Selecting Data:
			- ```
			  -- Verify that the table is empty by running the following command:
			  SELECT * FROM company_metadata LIMIT 10;
			  ```
		-
	- Load SEC filings from a semi-structured JSON format.
	- Use the Snowflake Marketplace to find free stock price data from Cybersyn.
- Uploading JSON
	- Cloning Tables
		- Zero Copy Clone means, takes snapshot of the data available, and makes it available by copying the schema and linking the data, this avoids duplicated data.
		- ```sql
		  CREATE TABLE company_metadata_dev CLONE company_metadata;
		  ```
	- Create tables:
		- ```sql
		  CREATE TABLE sec_filings_index (v variant);
		  - CREATE TABLE sec_filings_attributes (v variant);
		  ```
	- Create a stage:
		- ```sql
		  CREATE STAGE cybersyn_sec_filings
		  url = 's3://sfquickstarts/zero_to_snowflake/cybersyn_cpg_sec_filings/';
		  LIST @cybersyn_sec_filings;
		  ```
	- Strip and load the data:
		- ```sql
		  COPY INTO sec_filings_index
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_index.json.gz
		    file_format = (type = json strip_outer_array = true);
		  - COPY INTO sec_filings_attributes
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_attributes.json.gz
		    file_format = (type = json strip_outer_array = true);
		  ```
		- Here you can see the names that appears when selecting
	- LIMITING
		- ```sql
		  COPY INTO sec_filings_index
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_index.json.gz
		    file_format = (type = json strip_outer_array = true);
		  - COPY INTO sec_filings_attributes
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_attributes.json.gz
		    file_format = (type = json strip_outer_array = true);
		  ```
	- Changing Views:
		- ```sql
		  CREATE OR REPLACE VIEW sec_filings_index_view AS
		  SELECT
		    v:CIK::string                   AS cik,
		    v:COMPANY_NAME::string          AS company_name,
		    v:EIN::int                      AS ein,
		    v:ADSH::string                  AS adsh,
		    v:TIMESTAMP_ACCEPTED::timestamp AS timestamp_accepted,
		    v:FILED_DATE::date              AS filed_date,
		    v:FORM_TYPE::string             AS form_type,
		    v:FISCAL_PERIOD::string         AS fiscal_period,
		    v:FISCAL_YEAR::string           AS fiscal_year
		  FROM sec_filings_index;
		  - CREATE OR REPLACE VIEW sec_filings_attributes_view AS
		  SELECT
		    v:VARIABLE::string            AS variable,
		    v:CIK::string                 AS cik,
		    v:ADSH::string                AS adsh,
		    v:MEASURE_DESCRIPTION::string AS measure_description,
		    v:TAG::string                 AS tag,
		    v:TAG_VERSION::string         AS tag_version,
		    v:UNIT_OF_MEASURE::string     AS unit_of_measure,
		    v:VALUE::string               AS value,
		    v:REPORT::int                 AS report,
		    v:STATEMENT::string           AS statement,
		    v:PERIOD_START_DATE::date     AS period_start_date,
		    v:PERIOD_END_DATE::date       AS period_end_date,
		    v:COVERED_QTRS::int           AS covered_qtrs,
		    TRY_PARSE_JSON(v:METADATA)    AS metadata
		  FROM sec_filings_attributes;
		  ```
- SNOWFLAKE Internal SQL Queries
  collapsed:: true
	- Compute Statistics
		- General Agregate Queries such as Percentages can be found here: https://docs.snowflake.com/en/sql-reference/functions-aggregation
	- Clone:
		- Cloning tables
		- ```sql
		  SELECT
		      meta.primary_ticker,
		      meta.company_name,
		      ts.date,
		      ts.value AS post_market_close,
		      (ts.value / LAG(ts.value, 1) OVER (PARTITION BY meta.primary_ticker ORDER BY ts.date) - 1)::DOUBLE AS daily_return,
		      AVG(ts.value) OVER (PARTITION BY meta.primary_ticker ORDER BY ts.date ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS five_day_moving_avg_price
		  FROM Financial__Economic_Essentials.cybersyn.stock_price_timeseries ts
		  INNER JOIN company_metadata meta
		  ON ts.ticker = meta.primary_ticker
		  WHERE ts.variable_name = 'Post-Market Close';
		  ```
	- Using Cache
		- Cached is done automatically.
		- Running the same cache, conserve the cache of the statistics being run,
		- Consider that the auto suspensions remains by default 10 minutes, which is also around the cache remaining.
		- ```sql
		  SELECT
		      meta.primary_ticker,
		      meta.company_name,
		      ts.date,
		      ts.value AS nasdaq_volume,
		      (ts.value / LAG(ts.value, 1) OVER (PARTITION BY meta.primary_ticker ORDER BY ts.date) - 1)::DOUBLE AS volume_change
		  FROM cybersyn.stock_price_timeseries ts
		  INNER JOIN company_metadata meta
		  ON ts.ticker = meta.primary_ticker
		  WHERE ts.variable_name = 'Nasdaq Volume';
		  ```
		-
	- joining Tables
		- Join: https://docs.snowflake.com/en/sql-reference/constructs/join
		- ```sql
		  WITH data_prep AS (
		      SELECT 
		          idx.cik,
		          idx.company_name,
		          idx.adsh,
		          idx.form_type,
		          att.measure_description,
		          CAST(att.value AS DOUBLE) AS value,
		          att.period_start_date,
		          att.period_end_date,
		          att.covered_qtrs,
		          TRIM(att.metadata:"ProductOrService"::STRING) AS product
		      FROM sec_filings_attributes_view att
		      JOIN sec_filings_index_view idx
		          ON idx.cik = att.cik AND idx.adsh = att.adsh
		      WHERE idx.cik = '0001637459'
		          AND idx.form_type IN ('10-K', '10-Q')
		          AND LOWER(att.measure_description) = 'net sales'
		          AND (att.metadata IS NULL OR OBJECT_KEYS(att.metadata) = ARRAY_CONSTRUCT('ProductOrService'))
		          AND att.covered_qtrs IN (1, 4)
		          AND value > 0
		      QUALIFY ROW_NUMBER() OVER (
		          PARTITION BY idx.cik, idx.company_name, att.measure_description, att.period_start_date, att.period_end_date, att.covered_qtrs, product
		          ORDER BY idx.filed_date DESC
		      ) = 1
		  )
		  
		  SELECT
		      company_name,
		      measure_description,
		      product,
		      period_end_date,
		      CASE
		          WHEN covered_qtrs = 1 THEN value
		          WHEN covered_qtrs = 4 THEN value - SUM(value) OVER (
		              PARTITION BY cik, measure_description, product, YEAR(period_end_date)
		              ORDER BY period_end_date
		              ROWS BETWEEN 4 PRECEDING AND 1 PRECEDING
		          )
		      END AS quarterly_value
		  FROM data_prep
		  ORDER BY product, period_end_date;
		  ```
	-
	-
- Using Time Travel #[[Time Travel]]
	- Working with role, Account Admin & Account Usage
	- Using Time travel:
		- Allows Undroping tables: https://quickstarts.snowflake.com/guide/getting_started_with_snowflake/index.html#8
		- Allows for Reversing Queries or selecting [[BEFORE]]
- [10. Working with Roles, Account Admin, & Account Usage](https://quickstarts.snowflake.com/guide/getting_started_with_snowflake/index.html#9)
	-
-
- Questions
	- How to load from private s3?
		- This is important as you want to load specifically there.