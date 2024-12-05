# Managing Data Growth and Improving Performance

WSO2 API Manager (WSO2 API-M) when in use, will store metadata and runtime data in its connected databases. For example, it stores APIs, applications, subscriptions, and tokens that are created by users. Metadata related to applications and APIs are not been written to the databases frequently. However, as runtime data depends on different attributes such as the number of users, number of connected applications, and usage patterns, having a considerable load on the system will result in runtime data accumulating slowly over time. This will result in high data growth of the tables and this in return will negatively impact the performance of the system. 

Invalid access tokens, revoked access tokens, Registry transaction-related logs, authorization codes, and user sessions are some of the runtime data that gets stored in these databases. When you carefully analyze this data, you will see that you do not always need to keep this data, other than for audit purposes. Therefore, this data could be cleaned up periodically.

WSO2 API Manager provides two methods to do the cleanup.
 
   -  [Regular cleaning](#regular-cleaning)
   -  [Deep cleaning](#deep-cleaning)
   
Regular cleanup is good for regular housekeeping which will slow down unused token growth. However, during peak windows, regular cleaning may be less effective due to potential query slowdowns.  

!!! warning
    The deep cleaning removes unused tokens, sessions, and registry data, and prevent the tables from continuously growing, which can improve performance in the production servers. Choose the appropriate cleanup method based on the server's traffic level.

## Regular Cleaning
This cleanup is done within the product. It cleans up unused token related data during the runtime. This is an event-based cleaning process that involves the cleaning of specific entries based on specific user actions. For example, when an access token is revoked, the token will be picked from the `IDN_OAUTH2_ACCESS_TOKEN` table and moved into the `IDN_OAUTH2_ACCESS_TOKEN_AUDIT` table to be audited. In addition to revoked tokens,  inactive and expired tokens also accumulate in this table. This table is not used by WSO2 API-M. These tokens are kept in the database for logging and audit purposes, but they can have a negative impact on the server's performance over time. Therefore, it is recommended to clean them.

!!! note
    From WSO2 API Manager 2.6.0 onwards it is configured by default to trigger token clean up during token generation, token refreshing, and token revocation. Therefore, when the state of the token (`TOKEN_STATE`) is changed during any of the latter mentioned processes for tokens that were in the `ACTIVE` state before, by default, such tokens will be removed from the `IDN_OAUTH2_ACCESS_TOKEN` table and stored in an audit table. Therefore, you don't need to manually clean up the unused tokens as guided below from API-M 2.6.0 onwards. However, you need to [configure WSO2 API Manager to perform regular cleaning](#configuring-wso2-api-m-to-perform-regular-cleaning).
   
### Configuring WSO2 API-M to perform regular cleaning

WSO2 API Manager triggers token cleanup during the following instances.

-   Token generation
-   Token refresh
-   Token revocation

!!! note
    The regular cleanup procedure is enabled by default, and it cleans up the **Inactive**, **Revoked**, and **Expired** tokens. If you disabled this procedure and after some time you want to enable this cleanup procedure, it is better to clean the access token table using the [deep cleaning](#deep-cleaning) process and thereafter enable the feature.

To enable or disable token cleanup, open the `<API-M_HOME>/repository/conf/deployment.toml` file and do the following changes. (Add the configuration if it does not exist in the `deployment.toml` file)

``` toml
[oauth.token_cleanup]
enable = true
retain_access_tokens_for_auditing = true
```

<table>
<colgroup>
<col width="30%" />
<col width="70%" />
</colgroup>
<tr class="even">
<th><b>Property</b></th>
<th><b>Description</b></th>
</tr>
<tr class="even">
<td>enable</td>
<td>Set this property to <code>true</code> to enable token cleanup. </br>Set this property to <code>false</code> to disable token cleanup.</td>
</tr>
<tr class="even">
<td>retain_access_tokens_for_auditing</td>
<td>Set this property to <code>true</code> to move the old, invalid tokens to the Audit table when token cleaning is enabled. </br>Set this property to <code>false</code> if you do not wish to store old tokens in the Audit table.</td>
</tr>
</table>


## Deep Cleaning
In this cleaning method, all the remaining token data, session data, and Registry data can be cleaned up using separate stored procedures for each type of data. The unused data is periodically analyzed and removed through a stored procedure that runs against the database. In the deep cleaning process, the validity of the data is checked in every record. If unused or old data is detected, the stored procedure will clean them. There are three stored procedures provided that could be used to do the following three cleanups.   

   - Token cleanup
   - Session cleanup 
   - Registry cleanup 
    

### Enabling deep cleaning (token, session, and Registry cleanup)

This will remove the old and invalid tokens, sessions, and Auth codes, which cannot be cleaned by the products inbuilt cleanup process.

!!! tip
    It is safe to run these steps in read-only mode or during a time when traffic on the server is low. However, the latter is not mandatory if you are evaluating or testing the product. However, if you are running a production deployment with a high volume of traffic, it is recommended that you run this periodically as the unused tokens, sessions count, etc. can be high.

1. Take a backup of the running database.
2. Optionally, set up the database dump in a test environment and test it for any issues.

     For more information on setting up a database dump, go to the [MySQL](https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#mysqldump-syntax), [SQL Server](https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/create-a-full-database-backup-sql-server), and [Oracle](https://docs.oracle.com/cd/E11882_01/backup.112/e10642/rcmbckba.htm#BRADV8138) official documentation.

    !!! tip
        WSO2 recommends testing the cleanup scripts and stored procedures before running or configuring them against your production database server.

3. Select the appropriate cleanup script, based on your database, from [here](https://github.com/wso2/carbon-identity-framework/tree/master/features/identity-core/org.wso2.carbon.identity.core.server.feature/resources/dbscripts/stored-procedures), and run it on the production database. 

     This takes a backup of the necessary tables, turns off SQL updates, and cleans the database of unused tokens. However, when running the script on production make sure the script has been tested at least once in a lower environment.
  
     Select the `token-cleanup` script to clean up the tokens, the `sessiondata-cleanup` script to cleanup the session data, and the `registry-cleanup` script to clean up the unused data in the Registry.

4. After the cleanup is over, start the WSO2 API Manager and test it thoroughly for any issues.

     You can also schedule a cleanup task that will automatically run after a given period of time as shown in the examples below:

    -   [**MySQL**](#schedule-task-for-mysql)
    -   [**SQL Server**](#schedule-task-for-sql-server)
 
    #### Schedule a cleanup task for MySQL
    ``` sql
    USE 'WSO2AM_DB';
    DROP EVENT IF EXISTS 'cleanup_tokens_event';
    CREATE EVENT 'cleanup_tokens_event'
    ON SCHEDULE
        EVERY 1 WEEK STARTS '2018-01-01 00:00.00'
    DO
        CALL 'WSO2AM_DB'.'cleanup_tokens'();
    -- 'Turn on the event_scheduler'
    SET GLOBAL event_scheduler = ON;

    ```
    #### Schedule a cleanup task for SQL Server

    ``` sql
    USE WSO2AM_DB ;
    GO
    -- Creates a schedule named CleanupTask.  
    -- Jobs that use this schedule execute every day when the time on the server is 01:00.  
    EXEC sp_add_schedule
    @schedule_name = N'CleanupTask' ,
    @freq_type = 4,
    @freq_interval = 1,
    @active_start_time = 010000 ;
    GO
    -- attaches the schedule to the job BackupDatabase
    EXEC sp_attach_schedule
    @job_name = N'BackupDatabase',
    @schedule_name = N'CleanupTask' ;
    GO
    ```

Replace `WSO2AM_DB` with the name of your WSO2 API Manager database in the above script.

## Registry database cleanup

Creating and updating APIs and adding tags and ratings cause different registry properties to be added to the Registry database (Registry DB) in WSO2 API-M. When `Registry Versioning` is enabled, older properties are persisted in the database leading to unnecessary record growth in the Registry tables in the database. This directly affects the performance of the product if the number of records increase beyond a certain amount. Due to this limitation, registry versioning is disabled by default in WSO2 API Manager 3.0.0 onwards to prevent unnecessary database growth.

In WSO2 API-M versions preceding 3.0.0 (2.x.x and 1.x.x), `Registry Versioning` is enabled by default, therefore you need to cleanup the Registry DB if there are large tables with millions of records.

You can cleanup the following tables in the Registry DB without affecting the functionality of the product.
-   REG_LOG
-   REG_PROPERTY
-   REG_RESOURCE_PROPERTY
-   REG_TAG
-   REG_RESOURCE_TAG

!!! info
    You can always take a backup of the Registry database before the cleanup as a precaution for any issues that might occur during cleanup.

!!! note "Before you begin..."
    Remember to shut down the server and stop any DB transactions before the database cleanup. This is to prevent any inconsistencies that could happen with new data that gets added to the DB during the cleanup.

The Cleanup scripts for the Registry DB tables are given below.

#### REG_LOG Table Cleanup

!!! example "DB Types: H2, DB2, MySQL, MSSQL, Oracle and Postgresql."
    ``` sql
    CREATE TABLE REG_LOG_IDS_TO_KEEP (
             REG_LOG_ID INTEGER,
             REG_TENANT_ID INTEGER
    );
    
    INSERT INTO REG_LOG_IDS_TO_KEEP (REG_LOG_ID, REG_TENANT_ID)
    SELECT MAX(REG_LOG_ID) AS REG_LOG_ID, REG_TENANT_ID FROM REG_LOG GROUP BY REG_PATH, REG_TENANT_ID;
    
    DELETE FROM REG_LOG WHERE REG_LOG_ID NOT IN (SELECT REG_LOG_ID FROM REG_LOG_IDS_TO_KEEP);
    DROP TABLE REG_LOG_IDS_TO_KEEP;
    
    DELETE FROM REG_LOG WHERE REG_ACTION = 7;
    ```



#### REG_PROPERTY and REG_RESOURCE_PROPERTY Table Cleanup

!!! example "DB types: H2, MSSQL and Postgresql"

    ``` sql
    CREATE TABLE TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID INTEGER);
    -- Extract resource property (ID) created when versioning is disabled --
    INSERT INTO TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID)
    SELECT REG_PROPERTY_ID
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_PATH_ID IN
        (SELECT REG_PATH_ID
            FROM REG_RESOURCE);
    -- Extract resource property (ID) created when versioning is enabled --
    INSERT INTO TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID)
    SELECT REG_PROPERTY_ID
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_VERSION IN
        (SELECT REG_VERSION
            FROM REG_RESOURCE);
    -- Drop the foreign key constraint --
    ALTER TABLE REG_RESOURCE_PROPERTY DROP CONSTRAINT REG_RESOURCE_PROPERTY_FK_BY_TAG_ID;
    -- delete all unwanted REG_RESOURCE_PROPERTY entries --
    DELETE
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_PROPERTY_ID NOT IN
        (SELECT REG_PROPERTY_ID
        FROM TEMP_REG_RESOURCE_PROPERTY_ID);
    -- delete all unwanted REG_PROPERTY entries --
    DELETE
    FROM REG_PROPERTY
    WHERE REG_ID NOT IN
        (SELECT REG_PROPERTY_ID
        FROM TEMP_REG_RESOURCE_PROPERTY_ID);
    -- Insert back the foreign key constraint --
    ALTER TABLE REG_RESOURCE_PROPERTY ADD CONSTRAINT REG_RESOURCE_PROPERTY_FK_BY_TAG_ID FOREIGN KEY (REG_PROPERTY_ID, REG_TENANT_ID) REFERENCES REG_PROPERTY (REG_ID, REG_TENANT_ID);
    -- drop temporary table --
    DROP TABLE TEMP_REG_RESOURCE_PROPERTY_ID;
    ```

!!! example "DB types: MySQL"

    ``` sql
    CREATE TABLE TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID INTEGER);
    -- Extract resource property (ID) created when versioning is disabled --
    INSERT INTO TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID)
    SELECT REG_PROPERTY_ID
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_PATH_ID IN
        (SELECT REG_PATH_ID
            FROM REG_RESOURCE);
    -- Extract resource property (ID) created when versioning is enabled --
    INSERT INTO TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID)
    SELECT REG_PROPERTY_ID
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_VERSION IN
        (SELECT REG_VERSION
            FROM REG_RESOURCE);
    -- Drop the foreign key constraint --
    ALTER TABLE REG_RESOURCE_PROPERTY DROP FOREIGN KEY REG_RESOURCE_PROPERTY_FK_BY_TAG_ID;
    -- delete all unwanted REG_RESOURCE_PROPERTY entries --
    DELETE
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_PROPERTY_ID NOT IN
        (SELECT REG_PROPERTY_ID
        FROM TEMP_REG_RESOURCE_PROPERTY_ID);
    -- delete all unwanted REG_PROPERTY entries --
    DELETE
    FROM REG_PROPERTY
    WHERE REG_ID NOT IN
        (SELECT REG_PROPERTY_ID
        FROM TEMP_REG_RESOURCE_PROPERTY_ID);
    -- Insert back the foreign key constraint --
    ALTER TABLE REG_RESOURCE_PROPERTY ADD CONSTRAINT REG_RESOURCE_PROPERTY_FK_BY_TAG_ID FOREIGN KEY (REG_PROPERTY_ID, REG_TENANT_ID) REFERENCES REG_PROPERTY (REG_ID, REG_TENANT_ID);
    -- drop temporary table --
    DROP TABLE TEMP_REG_RESOURCE_PROPERTY_ID;
    ```

!!! example "DB types: DB2 and Oracle"

    ``` sql
    CREATE TABLE TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID INTEGER);
    -- Extract resource property (ID) created when versioning is disabled --
    INSERT INTO TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID)
    SELECT REG_PROPERTY_ID
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_PATH_ID IN
        (SELECT REG_PATH_ID
            FROM REG_RESOURCE);
    -- Extract resource property (ID) created when versioning is enabled --
    INSERT INTO TEMP_REG_RESOURCE_PROPERTY_ID(REG_PROPERTY_ID)
    SELECT REG_PROPERTY_ID
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_VERSION IN
        (SELECT REG_VERSION
            FROM REG_RESOURCE);
    -- delete all unwanted REG_RESOURCE_PROPERTY entries --
    DELETE
    FROM REG_RESOURCE_PROPERTY
    WHERE REG_PROPERTY_ID NOT IN
        (SELECT REG_PROPERTY_ID
        FROM TEMP_REG_RESOURCE_PROPERTY_ID);
    -- delete all unwanted REG_PROPERTY entries --
    DELETE
    FROM REG_PROPERTY
    WHERE REG_ID NOT IN
        (SELECT REG_PROPERTY_ID
        FROM TEMP_REG_RESOURCE_PROPERTY_ID);
    -- drop temporary table --
    DROP TABLE TEMP_REG_RESOURCE_PROPERTY_ID;
    ```

#### REG_TAG and REG_RESOURCE_TAG Table Cleanup

!!! example "DB types: H2, MSSQL and Postgresql"

    ``` sql
    CREATE TABLE TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID INTEGER);
    -- Extract resource tag (ID) created when versioning is disabled --
    INSERT INTO TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID)
    SELECT REG_TAG_ID
    FROM REG_RESOURCE_TAG
    WHERE REG_PATH_ID IN
        (SELECT REG_PATH_ID
            FROM REG_RESOURCE);
    -- Extract resource tag (ID) created when versioning is enabled --
    INSERT INTO TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID)
    SELECT REG_TAG_ID
    FROM REG_RESOURCE_TAG
    WHERE REG_VERSION IN
        (SELECT REG_VERSION
            FROM REG_RESOURCE);
    -- Remove the foreign key constraint --
    ALTER TABLE REG_RESOURCE_TAG DROP CONSTRAINT REG_RESOURCE_TAG_FK_BY_TAG_ID;
    -- delete all unwanted REG_RESOURCE_TAG entries --
    DELETE
    FROM REG_RESOURCE_TAG
    WHERE REG_TAG_ID NOT IN
        (SELECT REG_TAG_ID
        FROM TEMP_REG_RESOURCE_TAG_ID);
    -- delete all unwanted REG_TAG entries --
    DELETE
    FROM REG_TAG
    WHERE REG_ID NOT IN
        (SELECT REG_TAG_ID
        FROM TEMP_REG_RESOURCE_TAG_ID);
    -- add the foreign key constraint back --
    ALTER TABLE REG_RESOURCE_TAG ADD CONSTRAINT REG_RESOURCE_TAG_FK_BY_TAG_ID FOREIGN KEY (REG_TAG_ID, REG_TENANT_ID) REFERENCES REG_TAG (REG_ID, REG_TENANT_ID);
    -- drop temporary table --
    DROP TABLE TEMP_REG_RESOURCE_TAG_ID;
    ```

!!! example "DB types: MySQL"

    ``` sql
    CREATE TABLE TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID INTEGER);
    -- Extract resource tag (ID) created when versioning is disabled --
    INSERT INTO TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID)
    SELECT REG_TAG_ID
    FROM REG_RESOURCE_TAG
    WHERE REG_PATH_ID IN
        (SELECT REG_PATH_ID
            FROM REG_RESOURCE);
    -- Extract resource tag (ID) created when versioning is enabled --
    INSERT INTO TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID)
    SELECT REG_TAG_ID
    FROM REG_RESOURCE_TAG
    WHERE REG_VERSION IN
        (SELECT REG_VERSION
            FROM REG_RESOURCE);
    -- Remove the foreign key constraint --
    ALTER TABLE REG_RESOURCE_TAG DROP FOREIGN KEY REG_RESOURCE_TAG_FK_BY_TAG_ID;
    -- delete all unwanted REG_RESOURCE_TAG entries --
    DELETE
    FROM REG_RESOURCE_TAG
    WHERE REG_TAG_ID NOT IN
        (SELECT REG_TAG_ID
        FROM TEMP_REG_RESOURCE_TAG_ID);
    -- delete all unwanted REG_TAG entries --
    DELETE
    FROM REG_TAG
    WHERE REG_ID NOT IN
        (SELECT REG_TAG_ID
        FROM TEMP_REG_RESOURCE_TAG_ID);
    -- add the foreign key constraint back --
    ALTER TABLE REG_RESOURCE_TAG ADD CONSTRAINT REG_RESOURCE_TAG_FK_BY_TAG_ID FOREIGN KEY (REG_TAG_ID, REG_TENANT_ID) REFERENCES REG_TAG (REG_ID, REG_TENANT_ID);
    -- drop temporary table --
    DROP TABLE TEMP_REG_RESOURCE_TAG_ID;
    ```

!!! example "DB types: DB2 and Oracle"

    ``` sql
    CREATE TABLE TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID INTEGER);
    -- Extract resource tag (ID) created when versioning is disabled --
    INSERT INTO TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID)
    SELECT REG_TAG_ID
    FROM REG_RESOURCE_TAG
    WHERE REG_PATH_ID IN
        (SELECT REG_PATH_ID
            FROM REG_RESOURCE);
    -- Extract resource tag (ID) created when versioning is enabled --
    INSERT INTO TEMP_REG_RESOURCE_TAG_ID(REG_TAG_ID)
    SELECT REG_TAG_ID
    FROM REG_RESOURCE_TAG
    WHERE REG_VERSION IN
        (SELECT REG_VERSION
            FROM REG_RESOURCE);
    -- delete all unwanted REG_RESOURCE_TAG entries --
    DELETE
    FROM REG_RESOURCE_TAG
    WHERE REG_TAG_ID NOT IN
        (SELECT REG_TAG_ID
        FROM TEMP_REG_RESOURCE_TAG_ID);
    -- delete all unwanted REG_TAG entries --
    DELETE
    FROM REG_TAG
    WHERE REG_ID NOT IN
        (SELECT REG_TAG_ID
        FROM TEMP_REG_RESOURCE_TAG_ID);
    -- drop temporary table --
    DROP TABLE TEMP_REG_RESOURCE_TAG_ID;
    ```