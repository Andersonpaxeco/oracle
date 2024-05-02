# Instructions
- Create account in https://apex.oracle.com/en/
- Request Workspace
- Run the following to create the data model

```sql
create table item(
    item varchar2(25) not null,
    dept number(4) not null,
    item_desc varchar2(25) not null
);

create table loc(
    loc number(10) not null,
    loc_desc varchar2(25) not null
);

create table item_loc_soh(
item varchar2(25) not null,
loc number(10) not null,
dept number(4) not null,
unit_cost number(20,4) not null,
stock_on_hand number(12,4) not null
);

/************************** ITEM 1 ****************************/
-- considering index on table item
create index item_idx on item(item);

-- considering index on table loc
create index loc_idx on loc(loc);

-- considering index on table item_loc_soh
create index item_loc_idx on item_loc_soh(item);

-- if item is unique -- adding PK on item table
ALTER TABLE item
ADD CONSTRAINT item_pk PRIMARY KEY (item);

-- if loc is unique -- adding PK on loc table
ALTER TABLE loc
ADD CONSTRAINT loc_pk PRIMARY KEY (loc);

-- also add fk in item and loc columns on table item_loc _soh
ALTER TABLE item_loc_soh
ADD CONSTRAINT fk_item
  FOREIGN KEY (item)
  REFERENCES item(item);

ALTER TABLE item_loc_soh
ADD CONSTRAINT fk_loc
  FOREIGN KEY (loc)
  REFERENCES loc(loc);

/************************** ITEM 2 ****************************

Some ideas :
    1 - Indexing: Proper indexing can significantly speed up data retrieval operations, especially for large tables. Analyze query patterns and create indexes on columns frequently used in WHERE clauses or JOIN conditions.
    2 - Partitioning: Partitioning divides large tables into smaller, more manageable parts, which can improve query performance by allowing the database to access only relevant partitions. Consider partitioning based on date ranges, regions, or other logical divisions.
    3 - Materialized Views: Materialized views store the results of a query as a physical table, allowing for faster access and reduced query processing time. They are particularly useful for complex queries or aggregations on large datasets.
    4 - Query Optimization: Review and optimize SQL queries to ensure they are written efficiently. Use EXPLAIN PLAN to analyze query execution paths and identify potential bottlenecks. Consider rewriting queries to use more efficient joins, filters, and indexes.
    5 - Bulk Processing: Utilize bulk processing techniques like bulk inserts, updates, and deletes to minimize transaction overhead and improve performance when dealing with large datasets.
    6 - Compression: Oracle offers various compression techniques to reduce storage space and improve I/O performance for large tables. Evaluate options such as Advanced Row Compression or Advanced Compression to optimize storage efficiency.
    7 - Database Statistics: Keep database statistics up-to-date to help the query optimizer make informed decisions about query execution plans. Regularly gather statistics on tables, indexes, and partitions using DBMS_STATS package or automatic statistics collection.
    8 - Tuning Parameters: Adjust database configuration parameters such as memory allocation, I/O settings, and parallel processing parameters to optimize performance for large data operations.
    9 - Use of Index-Organized Tables (IOT): For tables with primary key or unique key constraints and a small number of columns frequently queried together, consider using IOTs to improve access speed by storing the entire row within the index structure.
    10 - Data Archiving and Purging: Implement data archiving and purging strategies to remove obsolete or historical data from large tables, reducing the overall dataset size and improving query performance.
    11 - Caching: Utilize Oracle's caching mechanisms such as result cache, buffer cache, or SQL query result cache to store frequently accessed data in memory, reducing disk I/O overhead and improving response times.

*/
/********************* ITEM 3 ***********************************

    1 - Optimistic Locking: Use optimistic locking techniques such as version numbers or timestamps to manage concurrent access to rows. This approach allows multiple transactions to read and modify data simultaneously, only detecting conflicts during the update phase.
    2 - Partitioning: Partition large tables to distribute data across multiple physical segments. This can reduce contention by allowing concurrent transactions to operate on different partitions simultaneously, minimizing contention on individual rows.
    3 - Indexing: Ensure that appropriate indexes are in place to support queries and enforce uniqueness constraints without causing excessive contention. However, be cautious with indexes on frequently updated columns, as they can increase contention due to index maintenance overhead.
*/

/************************** ITEM 4 ****************************/

CREATE VIEW v_item_loc_soh (item, loc, dept)
	AS SELECT item, loc, dept 
    from item, loc ;

/************************** ITEM 5 ****************************/
 create table dept(
    dept number(4) not null,
    dept_desc varchar2(25) not null,
    user_id number(10) not null
);


create index dept_idx on dept(dept);
-- or --
ALTER TABLE dept
ADD CONSTRAINT dept_pk PRIMARY KEY (dept);

/************************** ITEM 6 ****************************/

CREATE OR REPLACE PACKAGE store_management_pkg AS
    -- Procedure to save item_loc_soh data to a new table with stock value
    PROCEDURE save_item_loc_soh(
        p_store_id IN NUMBER
    );
END store_management_pkg;
/


CREATE OR REPLACE PACKAGE BODY store_management_pkg AS
    -- Procedure to save item_loc_soh data to a new table with stock value
    PROCEDURE save_item_loc_soh(
        p_store_id IN NUMBER
    ) AS
    BEGIN
        -- Insert data into the new table with calculated stock value
        INSERT INTO item_loc_soh_plus (item_id, location_id, stock_on_hand, unit_cost, stock_value)
        SELECT item_id, location_id, stock_on_hand, unit_cost, unit_cost * stock_on_hand AS stock_value
        FROM item_loc_soh
        WHERE item_id = p_store_id;
        COMMIT;
    END save_item_loc_soh;
END store_management_pkg;
/


/************************** ITEM 7 ****************************/


CREATE OR REPLACE PROCEDURE filter_data_by_dept(
    p_user_id IN NUMBER
) AS
    v_sql_query VARCHAR2(4000);
BEGIN
    -- Construct SQL query based on user's department association
    v_sql_query := 'SELECT * FROM dept WHERE ';
    
    -- Add department filter based on user ID
    IF p_user_id = 1 THEN -- Example condition, replace with your logic
        v_sql_query := v_sql_query || 'user_id = 1';
    ELSIF p_user_id = 2 THEN
        v_sql_query := v_sql_query || 'user_id = 2';
    ELSE
        -- Default filter for users not associated with any department
        v_sql_query := v_sql_query || '1 = 0'; -- No rows returned
    END IF;

    -- Execute the dynamic SQL query
    EXECUTE IMMEDIATE v_sql_query;
END filter_data_by_dept;
/

/************************** ITEM 8 ****************************/

-- Create a nested table type
CREATE OR REPLACE TYPE location_list_type AS TABLE OF VARCHAR2(100);

-- Create the pipeline function
CREATE OR REPLACE FUNCTION get_location_list RETURN location_list_type PIPELINED IS
BEGIN
    -- You can replace this query with any logic to fetch locations
    FOR location_rec IN (SELECT dept FROM dept) LOOP
        PIPE ROW(location_rec.location_name);
    END LOOP;
    RETURN;
END;
/

/************************** ITEM 9 ****************************

In this plan we have a full access in table with more than 10k lines!

------------------------------------------------------------------------------
| Id  | Operation           | Name         | Rows | Bytes | Cost  | Time       |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |              | 100019 | 40760 | 10840 | 00:00:03 |
| * 1 |   TABLE ACCESS FULL | ITEM_LOC_SOH | 100019 | 40760 | 10840 | 00:00:03 |
------------------------------------------------------------------------------

To optimize this query we put an index on some fields to optmize and reduce the cost and running statistics!

Plan hash value: 1697218418

----------------------------------------------------------------------------------
| Id  | Operation         | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |              |    11M|   702M| 11116   (2)| 00:00:01 |
|   1 |  TABLE ACCESS FULL| ITEM_LOC_SOH |    11M|   702M| 11116   (2)| 00:00:01 |
----------------------------------------------------------------------------------

Plan hash value: 1633205274

-------------------------------------------------------------------------------------
| Id  | Operation            | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |              |    11M|   149M|  6152   (2)| 00:00:01 |
|   1 |  INDEX FAST FULL SCAN| ITEM_LOC_IDX |    11M|   149M|  6152   (2)| 00:00:01 |
-------------------------------------------------------------------------------------

*/

--- in average this will take 1s to be executed
insert into item(item,dept,item_desc)
select level, round(DBMS_RANDOM.value(1,100)), translate(dbms_random.string('a', 20), 'abcXYZ', level) from dual connect by level <= 500 --10000;

--- in average this will take 1s to be executed
insert into loc(loc,loc_desc)
select level+100, translate(dbms_random.string('a', 20), 'abcXYZ', level) from dual connect by level <= 50 --1000;

-- in average this will take less than 120s to be executed
insert into item_loc_soh (item, loc, dept, unit_cost, stock_on_hand)
select item, loc, dept, (DBMS_RANDOM.value(5000,50000)), round(DBMS_RANDOM.value(1000,100000))
from item, loc;

commit;
```
- in the Apex App Builder import the application StockApplication.sql. When opening the application for login will be the same email and password as registered at apex.oracle.com.


**NOTE: If you fail to complete any challenge please still reply with what was the thinking and why you were not able to complete.**


[INFO]
> After finishing the problems, create a public repository in Github, push your code and send us the Github Public URL.
> If for some reason this is not possible, send us a zip folder containing a install script with all the required solution identified.
> The source contains a StockApplication.zip Apex application that can be deployed to better understand the challenge from user perspective.

# Context
Item loc stock an hand represents a snapshot table of stock in a specific moment for all items in all stores/warehouses for a retailer. In scenario where you have an Apex application that enables a view of stock per store/warehouse please consider the following:
 - this application has an very high user concurrency access during the entire day
 - the access to the application data is per store/warehouse
 - one of the attributes that most store/warehouse users search is by dept
 
# Challenge
## Must Have
### Data Model
Please consider that your reply for each point should include an explanation and corresponding sql code 
1. Primary key definition and any other constraint or index suggestion
2. Your suggestion for table data management and data access considering the application usage, for example, partition...
3. Your suggestion to avoid row contention at table level parameter because of high level of concurrency
4. Create a view that can be used at screen level to show only the required fields
5. Create a new table that associates user to existing dept(s)

### PLSQL Development
6. Create a package with procedure or function that can be invoked by store or all stores to save the item_loc_soh to a new table that will contain the same information plus the stock value per item/loc (unit_cost*stock_on_hand)
7. Create a data filter mechanism that can be used at screen level to filter out the data that user can see accordingly to dept association (created previously)
8. Create a pipeline function to be used in the location list of values (drop down)

## Should Have
### Performance
9. Looking into the following explain plan what should be your recommendation and implementation to improve the existing data model. Please share your solution in sql and the corresponding explain plan of that solution. Please take in consideration the way that user will use the app.
```sql

 Plan Hash Value  : 1697218418 

------------------------------------------------------------------------------
| Id  | Operation           | Name         | Rows | Bytes | Cost  | Time       |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |              | 100019 | 40760 | 10840 | 00:00:03 |
| * 1 |   TABLE ACCESS FULL | ITEM_LOC_SOH | 100019 | 40760 | 10840 | 00:00:03 |
------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 1 - filter("LOC"=652 AND "DEPT"=68)


Notes
-----
- Dynamic sampling used for this statement ( level = 2 )

```


 10. Run the previous method that was created on 6. for all the stores from item_loc_soh to the history table. The entire migration should not take more than 10s to run (don't use parallel hint to solve it :)) 
 11. Please have a look into the AWR report (AWR.html) in attachment and let us know what is the problem that the AWR is highlighting and potential solution.

## Nice to have
### Performance
11. Create a program (plsql and/or java, or any other language) that can extract to a flat file (csv), 1 file per location: the item, department unit cost, stock on hand quantity and stock value.
Creating the 1000 files should take less than 30s.
 
 

