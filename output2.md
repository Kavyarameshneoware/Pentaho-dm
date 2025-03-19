<html><body><pre>```text
## Pentaho Transformation Documentation: tx_load_contract

### 1. Overview

This Pentaho transformation, named `tx_load_contract`, is designed to load contract data from a CSV file, validate the `accountid` against a staging database, and then insert or update the corresponding contract records into the Salesforce contract table within the staging database. If the `accountid` is invalid, the transformation captures the error and writes the invalid record to an error table.

### 2. Data Flow

The transformation follows these steps:

1.  **ip_contract_data (CSV Input):** Reads contract data from a specified CSV file.

2.  **dblkp_account (DB Lookup):** Looks up the `accountid` in the `sfdc.account` table of the `pgsql_stagingdb` database.

3.  **fx_load_tmstmp (Formula):** Adds audit fields to the data stream, including a timestamp (`load_timestamp`) and the ETL process user (`var_create_update_by`).

4.  **swc_accountid (Switch / Case):** Routes the data based on the result of the `accountid` lookup.
    *   If `lkp_account_id` is null (account not found), the data is routed to the error handling steps.
    *   If `lkp_account_id` is not null (account found), the data is routed to the insert/update step.

5.  **insupd_contract (Insert / Update):** Inserts or updates the contract data into the `sfdc.contract` table in the `pgsql_stagingdb` database.

6.  **const_err_fields (Constant):** Defines constant error fields if `accountid` is not found.

7.  **err_contract (Table Output):** Writes records that failed insert/update (due to any error other than the `accountid` lookup failure). to the `stg_sfdc_err.contract_err` table in the `pgsql_errordb` database.

8.  **err_contract_2 (Table Output):** Writes records with invalid `accountid` values to the `stg_sfdc_err.contract_err` table in the `pgsql_errordb` database.

### 3. Configuration and Execution

*   **Input File Path:**  The transformation relies on the variables `${var_base_dir_path}`, `${var_batch_id}`.
    *   `${var_base_dir_path}`: This is the base directory for the ETL processes.
    *   `${var_batch_id}`: This is a unique identifier for the current batch process.
    *   Make sure these two variables are available when launching the transformation or the file cannot be located.

*   **Database Connections:** The transformation relies on three pre-defined database connections: `pgsql_stagingdb`, `pgsql_errordb`, and `pgsql_auditlogs` defined in the `kettle.properties` file or through the Pentaho environment. These connections need to be properly configured with the correct host, port, database name, username, and password.

*   **Execution:**
    1.  Ensure the CSV input file exists at the specified location defined by `${var_base_dir_path}` and `${var_batch_id}`.
    2.  Verify the database connections are correctly configured and the target tables exist in the `staging_db` and `error_db` databases.
    3.  Run the transformation in Pentaho Data Integration (PDI).
    4.  Monitor the transformation logs for any errors.

### 4. Detailed Survey

*   **Error Handling:** The transformation includes error handling to capture records that fail during the insert/update process. These records are written to an error table for further investigation.  The switch/case step allows explicit handling for invalid `accountid` and write into the errordb.

*   **Database Lookup:**  The `DB Lookup` step is crucial for ensuring data integrity by validating the `accountid` against the staging database. This helps prevent foreign key constraint violations during the insert/update process.  The transformation uses a "Lookup" step (dblkp_account) to enrich the input record with information from the Account table based on the Account ID in the input file. This step configuration can have significant impact on data flow and performance, if not planned correctly.

*   **Audit Fields:** The `Formula` step adds audit fields to track the ETL process and provide information on when and by whom the records were processed.  These fields are crucial for debugging and auditing purpose in the future.

### 5. Database Connections and Configuration

The transformation utilizes the following database connections:

*   **pgsql_stagingdb:**
    *   **Type:** PostgreSQL
    *   **Server:** smidatastore-prod.cluster-cxwm6k84w1hz.us-west-2.rds.amazonaws.com
    *   **Database:** staging_db
    *   **Port:** 5432
    *   **Username:** pdi_svc
    *   **Password:** Encrypted (stored in the configuration)
    *   **Purpose:** Used for looking up account details and inserting/updating contract data into the `sfdc.contract` table.

*   **pgsql_errordb:**
    *   **Type:** PostgreSQL
    *   **Server:** smidatastore-prod.cluster-cxwm6k84w1hz.us-west-2.rds.amazonaws.com
    *   **Database:** error_db
    *   **Port:** 5432
    *   **Username:** pdi_svc
    *   **Password:** Encrypted (stored in the configuration)
    *   **Purpose:** Used to write error records to the `stg_sfdc_err.contract_err` table.

*   **pgsql_auditlogs:**
     *   **Type:** PostgreSQL
     *   **Server:** smidatastore-prod.cluster-cxwm6k84w1hz.us-west-2.rds.amazonaws.com
     *   **Database:** audit_logs
     *   **Port:** 5432
     *   **Username:** pdi_svc
     *   **Password:** Encrypted (stored in the configuration)
     *   **Purpose:** Although the transformations has this database connection but it is not used in any steps.

### 6. Transformation Steps and Details

| Step Name | Type | Function | Key Details |
|-----------|------|----------|-------------|
| ip_contract_data | CSV Input | Reads contract data from a CSV file | - **Filename:** `${var_base_dir_path}/staging/edp/${var_batch_id}/v2_l2sfdc_pentaho_contract_${var_batch_id}.csv`  - **Separator:** Comma (,) - **Enclosure:** Double quote (") - **Header:** Yes (Y) - Specifies the fields and data types of the CSV file. |
| dblkp_account | DB Lookup | Looks up account details in the staging database | - **Connection:** `pgsql_stagingdb` - **Schema:** `sfdc` - **Table:** `account` - **Lookup Key:**  `accountid` = `id` (from the account table) - **Return Value:** `id` (renamed to `lkp_account_id`) from the account table. This is to validate `accountid`. |
| fx_load_tmstmp | Formula | Adds audit fields to the data stream | - Adds `load_timestamp` (current timestamp) and `var_create_update_by` ("pentaho"). |
| swc_accountid | Switch / Case | Routes data based on account ID lookup | - **Fieldname:** `lkp_account_id` - If `lkp_account_id` is null, routes to `const_err_fields`. - Otherwise, routes to `insupd_contract`. |
| insupd_contract | Insert / Update | Inserts or updates contract data in the staging database | - **Connection:** `pgsql_stagingdb` - **Schema:** `sfdc` - **Table:** `contract` - **Lookup Key:** `id` = `id` (from the incoming stream) - Specifies the fields to be updated if a matching record is found. |
| const_err_fields | Constant | Defines constant error fields when Account ID is not found | - Defines `err_nrfield`, `err_description`, `err_fieldname`, and `err_code` with values related to foreign key constraint violation for `accountid`. |
| err_contract | Table Output | Writes error records to the error table | - **Connection:** `pgsql_errordb` - **Schema:** `stg_sfdc_err` - **Table:** `contract_err` - Inserts all input fields into the table. - Catches database errors that occur when the insert/update fails for some other reason |
| err_contract_2 | Table Output | Writes invalid account ID records to the error table | - **Connection:** `pgsql_errordb` - **Schema:** `stg_sfdc_err` - **Table:** `contract_err` - Writes records when accountId lookup fails. |
```</pre></body></html>