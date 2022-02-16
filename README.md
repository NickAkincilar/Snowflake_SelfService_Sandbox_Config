# Snowflake SelfService Sandbox Configuration.

![alt text](https://github.com/NickAkincilar/Snowflake_SelfService_Sandbox_Config/blob/6db947f2051f52262c8cdae0fd421c40a5fd3ffc/Diagram.png)


Following Script creates a Self service Sandbox enviroment within Snowflake where users gain access to unique schemas where they have full rights to upload, create, modify & delete their data objects but restricted in terms of sharing their content with others. Part of this setup is also to grant them read-only access to other governed datasets within the account for them to join their own data.

``` sql
-- ******* 2/16/2022 by Nick Akincilar ****
-- SCRIPT TO CREATE A SELF SERVICE ENVIRONMENT FOR USERS
-- THIS SCRIPT CREATES THE FOLLOWING:
--
-- 1. A dedicated virtual warehouse : SELF_SERVICE_WH 
-- 2. Monthly cost monitoring for the SELF_SERVICE_WH Warehouse
-- 3. A database named SANDBOX_DB to be shared by all self-service users
-- 4. STORED PROCS: 
--    * - ADD_SANDBOX_USER(user_name varchar)               --> Enrolls an existing user to sandbox environment
--    * - ADD_SANDBOX_ROLE(new_role varchar, uid varchar)   --> Created a role per each user to limit them to single schema 
-- 5. Generic role (SANDBOX_ROLE) which will be used to grant read rights to governed datasets. 
-- 
-- NOTES: 
-- Sandbox environment is a separate database using managed schemas per user. 
-- Each user gets access to their own schema where they will have full rights to add/create/drop tables but they won't be
-- able to grant rights to other users to their content. Any grants to content created in these schemas will have to be done by
-- a higher level admin. 
-- For managing & removing unused user created content, tables & views that have not been accessed in the last X days can be 
-- identified via SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY (Enterprise edition & above) and be automatically removed.
-- 

-- ************ RUN THIS TO CREATE THE FRAMEWORK *******

USE ROLE SYSADMIN;

CREATE OR REPLACE DATABASE SANDBOX_DB;

-- CREATE A DEDICATED COMPUTE CLUSTER FOR SELF-SERVICE SANDBOX.

CREATE OR REPLACE WAREHOUSE SELF_SERVICE_WH 
WITH 
  WAREHOUSE_SIZE = 'SMALL' 
  WAREHOUSE_TYPE = 'STANDARD' 
  AUTO_SUSPEND = 600 
  AUTO_RESUME = TRUE 
  MIN_CLUSTER_COUNT = 1 
  MAX_CLUSTER_COUNT = 3 
  SCALING_POLICY = 'STANDARD';



-- CREATE A CREDIT MONITOR FOR VIRTUAL WAREHOUSE (500 credits per month)

USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE RESOURCE MONITOR "SANDBOX_COST_MONITOR" WITH CREDIT_QUOTA = 500 
 TRIGGERS 
 ON 100 PERCENT DO SUSPEND 
 ON 75 PERCENT DO NOTIFY;
 
ALTER WAREHOUSE SELF_SERVICE_WH SET RESOURCE_MONITOR = "SANDBOX_COST_MONITOR";


USE ROLE SECURITYADMIN;
CREATE OR REPLACE ROLE SANDBOX_ROLE;
GRANT USAGE ON DATABASE SANDBOX_DB TO ROLE SANDBOX_ROLE;


USE ROLE SYSADMIN;

GRANT USAGE ON WAREHOUSE SELF_SERVICE_WH TO ROLE SANDBOX_ROLE;
GRANT USAGE  ON DATABASE SANDBOX_DB TO ROLE SECURITYADMIN;
GRANT ALL  ON WAREHOUSE SELF_SERVICE_WH TO ROLE SECURITYADMIN;
GRANT ALL ON SCHEMA SANDBOX_DB.PUBLIC TO ROLE SECURITYADMIN;



USE ROLE SECURITYADMIN;
USE SCHEMA SANDBOX_DB.PUBLIC;
  
  create or replace procedure ADD_SANDBOX_ROLE(new_role varchar, uid varchar)
  returns varchar
  language sql
  as
  $$

    begin

      LET nuid string := '"' || :uid || '"';
      LET UC integer := 1;
     
      
      if (UC > 0) then
        CREATE OR REPLACE ROLE identifier(:new_role);
        GRANT ROLE identifier(:new_role) TO USER identifier(:nuid) ;    
        return :new_role || ' role is created';
      else
        return 'No user found';
      end if;  
    end;
  $$  ;
  
  
GRANT USAGE ON PROCEDURE IDENTIFIER('SANDBOX_DB.PUBLIC.ADD_SANDBOX_ROLE') (VARCHAR, VARCHAR) TO ROLE IDENTIFIER('SYSADMIN');



USE ROLE SYSADMIN;
USE SCHEMA SANDBOX_DB.PUBLIC;

create or replace procedure ADD_SANDBOX_USER(user_name varchar)
  returns varchar
  language sql
  as
  $$

    begin
      LET uid := :user_name;
      user_name :=  REPLACE(:user_name, '@', '_');
      user_name :=  REPLACE(:user_name, '.', '_');
      user_name :=  REPLACE(:user_name, '#', '_');    
 
      LET new_schema := 'SANDBOX_DB.' || user_name || '_dev'; 
      LET new_role := 'SANDBOX_' || user_name;

      CREATE OR REPLACE SCHEMA identifier(:new_schema)  WITH MANAGED ACCESS;
      
      call ADD_SANDBOX_ROLE(:new_role, :uid);
      
      GRANT USAGE ON DATABASE SANDBOX_DB TO ROLE identifier(:new_role);
      GRANT ALL ON SCHEMA identifier(:new_schema) TO ROLE identifier(:new_role);
      GRANT USAGE ON WAREHOUSE SELF_SERVICE_WH TO ROLE identifier(:new_role);
      
      return :new_schema || ' sandbox is created';
    end;
  $$  
  ;
  
  
 
 
 -- ************ ONCE THE FRAMEWORK IS CONFIGURED, USE THE FOLLOWING TO ENROLL USERS & REMOVE UNUSED DATA *******
 
 --- EXECUTE FOLLOWING TO ENROLL A USER INTO THE SANDBOX ENVIRONMENT ( Creates a dedicated schema & a role for this user) 
 --- USER ID IS CASE SENSITIVE
 
USE ROLE SYSADMIN;
USE SCHEMA SANDBOX_DB.PUBLIC;

call ADD_SANDBOX_USER('JOHN');


-- Following Query allows to identify tables & views that have not been accessed in the last 90 days and creates the neessary 
-- SQL commands to drop them in order to remove unused content from the sandbox environment
-- This requires Enterprise edition or higher

use role accountadmin;
use schema snowflake.account_usage;

-- Tables not being queries in last 90
with ObjectsCurrent as
(
select 

  f1.value:"objectName"::string as BaseObjectName,
  f1.value:"objectDomain"::string as BaseObjectType,
  f2.value:"objectName"::string as ObjectName,
  f2.value:"objectDomain"::string as ObjectType  

from access_history
, lateral flatten(base_objects_accessed) f1
,lateral flatten(direct_objects_accessed) f2  

where
(
 f1.value:"objectDomain"::string='Table'
OR 
f2.value:"objectDomain"::string='View'
 )
 
and query_start_time >= dateadd('day', -90, current_timestamp())
group by 1,2,3,4),

ObjectsOld as
(
select 

  f1.value:"objectName"::string as BaseObjectName,
  f1.value:"objectDomain"::string as BaseObjectType,
  f2.value:"objectName"::string as ObjectName,
  f2.value:"objectDomain"::string as ObjectType

from access_history
, lateral flatten(base_objects_accessed) f1
,lateral flatten(direct_objects_accessed) f2  
where  
(
 f1.value:"objectDomain"::string='Table'
OR 
f2.value:"objectDomain"::string='View'
  
 )

and query_start_time <= dateadd('day', -90, current_timestamp())
group by 1,2,3,4)

SELECT OBJECTNAME as Name, OBJECTTYPE, 'DROP ' || OBJECTTYPE || ' ' || OBJECTNAME || ';' as SQL FROM ObjectsOld WHERE 
BaseObjectName NOT IN (SELECT BaseObjectName FROM ObjectsCurrent)
AND BaseObjectName LIKE 'SANDBOX_DB.%'
AND OBJECTTYPE = 'View'

UNION

SELECT BaseObjectName as Name, OBJECTTYPE , 'DROP ' || OBJECTTYPE || ' ' || BaseObjectName || ';' as SQL FROM ObjectsOld WHERE 
BaseObjectName NOT IN (SELECT BaseObjectName FROM ObjectsCurrent)
AND BaseObjectName LIKE 'SANDBOX_DB.%'
AND OBJECTTYPE = 'Table'

``` 
