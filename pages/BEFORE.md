- ```sql
  -- Set the session variable for the query_id
  SET query_id = (
    SELECT query_id
    FROM TABLE(information_schema.query_history_by_session(result_limit=>5))
    WHERE query_text LIKE 'UPDATE%'
    ORDER BY start_time DESC
    LIMIT 1
  );
  -- Use the session variable with the identifier syntax (e.g., $query_id)
  CREATE OR REPLACE TABLE company_metadata AS
  SELECT *
  FROM company_metadata
  BEFORE (STATEMENT => $query_id);
  -- Verify the company names have been restored
  SELECT *
  FROM company_metadata;
  ```
- From: https://quickstarts.snowflake.com/guide/getting_started_with_snowflake/index.html#8
- ```sql
  UPDATE company_metadata SET company_name = 'oops';
  SELECT *
  FROM company_metadata;
  ```
	- ![one row result](https://quickstarts.snowflake.com/guide/getting_started_with_snowflake/img/2a7ab14de8dde628.png)
-