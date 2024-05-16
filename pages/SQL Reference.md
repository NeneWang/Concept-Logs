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
	- Create tables:
		- ```
		  CREATE TABLE sec_filings_index (v variant);
		  - CREATE TABLE sec_filings_attributes (v variant);
		  ```
	- Create a stage:
		- ```
		  CREATE STAGE cybersyn_sec_filings
		  url = 's3://sfquickstarts/zero_to_snowflake/cybersyn_cpg_sec_filings/';
		  LIST @cybersyn_sec_filings;
		  ```
	- Strip and load the data:
		- ```
		  COPY INTO sec_filings_index
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_index.json.gz
		    file_format = (type = json strip_outer_array = true);
		  - COPY INTO sec_filings_attributes
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_attributes.json.gz
		    file_format = (type = json strip_outer_array = true);
		  ```
		- Here you can see the names that appears when selecting
	- LIMITING
		- ```
		  COPY INTO sec_filings_index
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_index.json.gz
		    file_format = (type = json strip_outer_array = true);
		  - COPY INTO sec_filings_attributes
		  FROM @cybersyn_sec_filings/cybersyn_sec_report_attributes.json.gz
		    file_format = (type = json strip_outer_array = true);
		  ```
	- Changing Views:
		- ```
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
	-
- Questions
	- How to load from private s3?