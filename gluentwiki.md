
- [gluent implementation](#gluent-implementation)
  * [advisor output](#advisor-output)
  * [installation doc](#installation-doc)
  * [Candidate Tables for Gluent Implementation sheet](#candidate-tables-for-gluent-implementation-sheet)
  * [pre-offload reports](#pre-offload-reports)
  * [top sql report per table](#top-sql-report-per-table)
  * [edb360 and sqld360 sql collection](#edb360-and-sqld360-sql-collection)
  * [performance testing runtime sheet](#performance-testing-runtime-sheet)
  * [app testing process](#app-testing-process)
    * [Table Rename and Synonym Creation Impact](#table-rename-and-synonym-creation-impact)
    * [dropping partitions impact](#dropping-partitions-impact)
  * [explaning gluent to developers](#explaning-gluent-to-developers)
- [aeg tools](#aeg-tools)
  * [general workflow](#general-workflow)
  * [aeg_toolkit - helper scripts for gluent administration](#aeg_toolkit)
  * [aeg_offload_tables - automate offloading of tables](#aeg_offload_tables) 
  * [get_run_stats - benchmark baselineperf and gluentperf](#get_run_stats)
  * [report_sqlmon - modified sqld360 for perf data collection](#report_sqlmon)
- [sql troubleshooting](#sql-troubleshooting)
  * [general workflow](#general-workflow-1)
    * [synonym to table translation](#synonym-to-table-translation)
  * [collect perf data](#collect-perf-data)
  * [troubleshooting steps](#troubleshooting-steps)  
  * [tips and tricks](#tips-and-tricks)
    * [gluent objects](#gluent-objects)
      * [fix parallelism on gluent objects](#fix-parallelism-on-gluent-objects)
      * [disable agg rewrite rule](#disable-agg-rewrite-rule)
      * [disable query rewrite](#disable-query-rewrite)
      * [enabling a rewrite rule](#enabling-a-rewrite-rule)
    * [remove USE_NL hint](#remove-use_nl-hint)
    * [no_rewrite hint](#no_rewrite-hint)
    * [present command](#present-command)
    * [rewrite query to enable predicate pushdown](#rewrite-query-to-enable-predicate-pushdown)
    * [parallelism should be the same for all gluent objects](#parallelism-should-be-the-same-for-all-gluent-objects)
    * [partition pruning helps in performance](#partition-pruning-helps-in-performance)
    * [between date SQLs](#between-date-sqls)
    * [be aware of processes that updates offloaded records](#be-aware-of-processes-that-updates-offloaded-records)
    * [on offloading performance](#on-offloading-performance)
    * [when impala query shows 1=1](#when-impala-query-shows-1-equals-1)
    * [when impala query shows 1 equals 1 + other binds](#when-impala-query-shows-1-equals-1-plus-other-binds)
    * [validate metad is working](#validate-metad-is-working)
    * [sqoop performance](#sqoop-performance)
      * [profile fix offload initial max SQL from hours to 3secs](#profile-fix-offload-initial-max-sql-from-hours-to-3secs)
      * [profile fix offload sqoop sql subpartitions null query](#profile-fix-offload-sqoop-sql-subpartitions-null-query)
      * [sqoop benchmarking](#sqoop-benchmarking)
      * [sqoop benchmark monitoring](#sqoop-benchmark-monitoring)
      * [get precision sampling remaining time](#get-precision-sampling-remaining-time)
      * [sqoop query performance bug](#sqoop-query-performance-bug)
    * [predicate push down by adding a join condition on filter key](#predicate-push-down-by-adding-a-join-condition-on-filter-key)
    * [troubleshoot query rewrite](#troubleshoot-query-rewrite)
    * [object OFFLOAD_BIN does not exist](#object-offload_bin-does-not-exist)
- [sql and command line cheatsheet](#sql-and-command-line-cheatsheet)
  * [gluent command line](#gluent-command-line)
    * [offload](#offload)
      * [automated offload](#automated-offload)
      * [how to know where data is split](#how-to-know-where-data-is-split)
    * [present](#present)
  * [command line](#command-line)
    * [check and restart metad](#check-and-restart-metad)
    * [get errors on conn log](#get-errors-on-conn-log)
    * [get memory from sqlmon files, get high memory SQL](#get-memory-from-sqlmon-files-get-high-memory-sql)
    * [get duration from sqlmon files](#get-duration-from-sqlmon-files)
    * [rename benchmark files](#rename-benchmark-files)
    * [move zip and planx files](#move-zip-and-planx-files)
  * [sqlplus](#sqlplus)
    * [show](#show)
    * [10053 trace](#10053-trace)
    * [looping synonyms and synonym chain](#looping-synonyms-and-synonym-chain)
    * [kill session](#kill-session)
  * [webgui](#webgui)
    * [query EXCEPTION SQLs on impala ui](#query-exception-sqls-on-impala-ui)
    * [show the plan tree and profile of an impala query](#show-the-plan-tree-and-profile-of-an-impala-query)
- [errors](#errors)
  * [ORA 29913 odciexttablefetch callout](#ora-29913-odciexttablefetch-callout)
  * [reread](#reread)
  * [ORA 600](#ora-600)
  * [memory limit exceeded](#memory-limit-exceeded)
  * [view error grant on offlaod](#view-error-grant-on-offlaod)
  * [cognos not null on view](#cognos-not-null-on-view)


<!-- toc -->

## gluent implementation

### advisor output
* this is where we identify the tables that are candidate for offload 

### installation doc
* hadoop/oracle specific install instructions for gluent

### Candidate Tables for Gluent Implementation sheet 
* from the advisor output, this is where we arrange/categorize the tables by application and other things like data retention, etc. etc.
* from here we can easily filter by app team, database, schema, tables, etc. which can be used for planning the offloading 

### pre-offload reports 
* from the data on the sheet above [Candidate Tables for Gluent Implementation sheet](#candidate-tables-for-gluent-implementation-sheet), we run the tool [aeg_offload_tables - automate offloading of tables](#aeg_offload_tables) to populate the table GLUENT_APP.AEG_OFFLOAD_TABLES with the columns below and we use the data on this table as inputs to offload parameters and automate the offloading of most of the tables 
	* OWNER
	* TABLE_NAME
	* GLUENT_TABLE_NAME
	* GB_TOTAL
	* DEGREE
	* TABLE_LAST_ANALYZED
	* PART_LAST_ANALYZED
	* PARTITIONED
	* SUB_PARTITIONED
	* PART_KEY
	* PART_KEY_COUNT
	* SUB_PART_KEY
	* SUB_PART_KEY_COUNT
	* DATA_TYPE
	* SUB_DATA_TYPE
	* INTERVAL
	* OLDER_THAN
	* ROW_COUNT_OLDER_THAN
	* ROW_COUNT_REMAINING
	* GB_OLDER_THAN
	* GB_REMAINING
	* PARTITION_COUNT_OLDER_THAN
	* PARTITION_COUNT_REMAINING

### top sql report per table 
* then while we evaluate or plan the offloading or while we do the offloading. we can do the performance assessment/collection of the SQLs and meet with the app teams on what SQLs are critical for performance testing 
* from the sheet [Candidate Tables for Gluent Implementation sheet](#candidate-tables-for-gluent-implementation-sheet) we get the table names for each database and use that as a filter for the SQL collector that captures topn SQL_IDs by: 
  * total elap
  * elap/exec 
  * exec 

Having three data set gives us more info about the profile of the SQL_IDs. The data we get have the following columns: 
 * DB                  
 * PSCHEMA             	
 * MODULE                                            	
 * OBJ_NAME                        	
 * SQL_ID       	
 * PLAN_HASH_VALUE	                        
 * FMS (force matching signature)	
 * SQL_TEXT            	  
 * ELAPEXEC <-- topn SQLID	     
 * EXECS <-- topn SQLID	     
 * ETIME <-- topn SQLID	   
 * CPUTIME	    
 * IOTIME	       
 * PIO	       
 * LIO	 
 * TIME_RANK	
 * SQLDETAIL

the data output above and the app team supplied SQLs should complete the scope of SQL perf testing.
this data will also serve as a driver for another tool which will parse the SQL_IDs to generate SQLD360 collection commands 

### edb360 and sqld360 sql collection 
* the data above and the excel sheet below serve as drivers to generate SQLD360 collection commands 
  * example raw scripts executed to generate collect and arrange sqls https://gist.github.com/karlarao/567de28d7547ab7f00cdf2f173c3f817
* each sqld360 is unzipped and the standalone SQL (with binds) is copied to be used for benchmarking the other data collected in sqld360 is used to backtrack/compare the performance of the SQL_ID in the prod environment if the gluent or baseline performance is slower. the metadata info is also useful to recreate the SQL_ID objects to investivate the query logic

### performance testing runtime sheet

the collection above and this sheet is part of the tool [get_run_stats - benchmark baselineperf and gluentperf](#get_run_stats) the general workflow of the tool are as follows: 
  * from the sheet [performance testing runtime sheet](#performance-testing-runtime-sheet) we drive what tables need performance testing 
  * collect SQLs based on list of tables
  * baseline the SQLs (baselineperf)
  * collect distinct tables baselined and compare with the list 
  * mark the tables baselined
  * any tables not baselined will go through another round of collection
  * tables that are baselined will go to next step "gluentperf" (run against hybridview) 
  * this goes on until all tables are marked as "gluentperf"
  * in summary the tables goes from collection->baselineperf->gluentperf 
  * in the process of running the SQLs, we also profile them and discover any issues and patterns that we can fix [tips and tricks](#tips-and-tricks). And we also use [report_sqlmon - modified sqld360 for perf data collection](#report_sqlmon) to standardize the performance data collection based on a SQL_ID that we can easily share with the gluent team
  * all performance runs are recorded in a table called GLUENT_APP.GET_RUN_STATS and from here we run a SQL to extract the test results and put it in excel for recording/reporting purposes to the client (pivot table/charts)

The GLUENT_APP.GET_RUN_STATS columns are as follows:
* TEST_NAME
* TEST_TYPE
* TEST_CATEGORY
* TEST_NOTES
* SNAP_TYPE
* SNAP_TIME
* STAT_CLASS
* NAME
* VALUE

On this table the following are collected: 
* script name used (TEST_NAME)
* Elapsed 
* CPU time
* SQL_ID
* Tables Accessed (this is critical and this shows us if only base table is used or both base and external table/gluent agg objects)
* Wait events

### app testing process 
While we do our own benchmark runs (baselineperf and gluentperf). The app team is also doing their own performance testing 

* Candidate tables loaded with one year of production data	
* Candidate tables renamed to add _GLT and synonyms created to point to the new table
  * ENS.BILLING_ACCOUNT renamed to ENS.BILLING_ACCOUNT_GLT
  * Synonym ENS.BILLING_ACCOUNT created to point to ENS.BILLING_ACCOUNT_GLT
* ETL load processes executed in DAILY map in DWTST to gather baseline and work any issues from the table rename/synonym creation
* Six months of data offloaded to Hadoop by Gluent.  (This data remains in Oracle for now)
* Table Synonyms re-created to point to the new Hybrid views.
  * Synonym ENS.BILLING_ACCOUNT now points to ENS_H.BILLING_ACCOUNT_GLT
* ETL load processes executing in DAILY map in DWTST to work through issues and compare runtimes

#### Table Rename and Synonym Creation Impact
Need to validate the following:
* Analyze table procedures
* Partition Swap procedures
* Truncate procedures (for tables that need to be mirrored on Oracle and Hadoop)
* Public synonyms

Failures in these cases -
   Table owner/name passed as parameters in job command or passed as parameters to another
	procedure or function
	
#### dropping partitions impact

Lessons Learned from TEST:  Dropping partitions in production can cause production job failures because indexes are invalid (need to address in maintenance windows)




### explaning gluent to developers
https://gist.github.com/karlarao/0b64f12b6a61f002a69c54e85e038465


## aeg tools 

### general workflow
* aeg_toolkit
* aeg_offload_tables 
* get_run_stats
* report_sqlmon 

### aeg_toolkit

### aeg_offload_tables

### get_run_stats

### report_sqlmon
the following are collected: 
* sqld360 sqlmonitor reports (all kinds of sqlmon report type)
* sqld360 metadata info (metadata info around the SQL_ID - similar to table_disco) 
* sqld360 standalone SQL (the SQL with binds that we can execute right away) 
* planx.sql (contains ASH, response time info, table, index and col stats, AWR_PLAN_CHANGE, etc.) 
* snapper.sql (based on the SQL_ID filter)
* gluent sql monitor (based on the SQL_ID filter)

all collected data are placed in folder dir_<SQL_ID> so we can easily tar the folder when we need to share with gluent team 


## SQL troubleshooting

### general workflow

#### synonym to table translation
```
(SYNONYM) ENS.SERVICE_AGREEMENT
	->> (VIEW) ENS_H.SERVICE_AGREEMENT_GLT
			->> (UNION ALL) 
					(BASE TABLE) "ENS"."SERVICE_AGREEMENT_GLT"
					(EXTERNAL TABLE) "ENS_H"."SERVICE_AGREEMENT_GLT_EXT"
```

### collect perf data

* @sqld360 \<sqlid\>
  * this will collect gluent sqlmon, sqld360 metadata, and planx 
* impala profile
* aeg log 
* db optimizer query logic visualization - only if absolutely needed 
* [10053 trace](#10053-trace) - only if absolutely needed

### troubleshooting steps

* planx
* ash plan_id 
* execution plan
* check gluent objects (external tables/agg objects) used and tables accessed 
* predicate info if filters/precates are getting pushed down 
* check impala queries generated 
* check imapala stats (duration and rows processed)
* check join conditions and query logic 
  * possible hints to alter the plan 
* check sql monitor (oracle and gluent)
* check metadata info for join keys that can be pushed down
  * possible rewrite 
  * possible additional join condition to allow push down 
* explore present command 


### tips and tricks

#### remove USE_NL hint 

* the SQL errors with "unable to reread file" 

#### no_rewrite hint 
/*+ no_rewrite */ 

* used in SQLs with "ROW_NUMBER() OVER(PARTITION BY" - this errors with ORA-600 


#### disable agg rewrite rule 
this can be done through the following: 
* hint /*+ no_rewrite */ 
* disabling the rewrite rule - [disable query rewrite](#disable-query-rewrite)

#### enabling a rewrite rule 
this can be done through the following: 
* by enabling the disabled rewrite rule
* by a custom rule through "present" command 
* only for a specific session
  * let's say you want to test the "effect" before implementing for real. You can do this by creating a rewrite rule and altering the category for that rule and in your session match the new category name let's say "TEST" with an alter session command
```
alter session set SQLTUNE_CATEGORY = 'TEST';
```
  * If we find a common query where a re-write would significantly help, we can evaluate re-enabling the rewrite or depending on the structure of the query, we can use gluent to present a custom rewrite that might only apply to that query or a few similar queries.  Another possibility is to set the rewrite to a specific category and have the session that needs the rewrite to set the category prior to running the query. We are leaving the count aggregation rewrite rules enabled.


#### present command  
* this also enables predicate pushdown. It took a while to create (40mins) because it's creating an aggregate data on the hadoop side and also another rewrite rule. But when the query used the new rewrite rule the query ran for 1sec from 536secs

[present](#present)

#### rewrite query to enable predicate pushdown
see example here https://gist.github.com/karlarao/131b1f1e35f2eaf8ac9538fe7696cb7a

#### parallelism should be the same for all gluent objects

#### partition pruning helps in performance

Try to use the table partitioned by column(s) in query where clauses for partition pruning when
possible.  In cases where tables partitioned by different columns are joined, the process may scan both oracle and Hadoop.
	
We had one table that was most often queried by LOAD_EXP_DATE but was partitioned by LOAD_DATE.  This caused queries to scan both Oracle and Hadoop.  We were able to  re-partition the table by LOAD_EXP_DATE without impacting the application, and performance improved.   

#### between date SQLs 

Time slicing records can cause the query to scan both Oracle and Hadoop
```
   	  p_load_date between osf.load_date and osf.load_exp_date
```

#### be aware of processes that updates offloaded records

Be aware of processes that can update an old record in cases where -
The source system updates or archives (deletes) old records 
A PK previously marked deleted returns active and current

Does your ETL attempt to update any columns on the old record?
			
Ensemble memo table is partitioned by sys_creation_date in DWPROD.  Ensemble application sometimes updates the sys_creation_date for an old MEMO_ID.  


#### on offloading performance

For big tables with many columns the initial sqoop query of the offload command may take a long time due to the estimation process. This can be made faster with a SQL Profile. The trick is to rewrite the query, remove the slow part of the query that messes up the query transformation, create a profile on the fast version, and then apply the Plan Hash Value of that to the slow SQL. 

And there are other things that can be done: 
1) parallelism/threads (64) 
2) fetch size (50K)
2) chunking (16GB max)
3) java heap size
4) offloading parameter to false (this is more on solving the mismatch)

#### when impala query shows 1 equals 1
when impala query shows 1=1 metad issue can't get bind values for a query 
metad process must be running under the root user
need to kill the one running as oracle
offload.env parameter setting that we had to set METAD_AUTOSTART=false it it's running under a cluster service 

#### when impala query shows 1 equals 1 plus other binds 
if case of on an inline-over-inline view you'll see 1=1 + other binds 
in this scenario you the binds/filters can't get pushed to the other view/subquery 

#### validate metad is working 

```
var test number; 
exec :test:=1; 

@find_object offload_dummy

select * from stage_h.offload_dummy where col1=:test; 

then go to the impala ui
and check the predicate 
col1=<predicate value> and bucket_id=<bucket number>
```

#### profile fix offload initial max SQL from hours to 3secs
```
We have the following SQL that ran long in DWTST that we fixed through SQL profile (from 50mins to 3secs). We are expecting this to run longer in PROD due to larger size table. 

SELECT MIN("LOAD_DATE") FROM "DIM"."ENS_CSM_SUMMARY_DT_GLT" 

I’ve attached the script to implement the fix. And please following the steps below: 

1)	On prod host 
            cd /db_backup_denx3/p1/gluent/karl
2)	Connect / as sysdba 
3)	Execute the script as follows 

@create_hint_sqlprofile
Enter value for sql_id: dg7zj0q9qa2gf
Enter value for profile_name (PROFILE_sqlid_MANUAL): <just hit ENTER here>
Enter value for category (DEFAULT): <just hit ENTER here>
Enter value for force_matching (false): <just hit ENTER here>
Enter value for hint_text: NO_PARALLEL
Profile PROFILE_dg7zj0q9qa2gf_MANUAL created.

This will make the query run in serial through a profile hint which is the fix for this issue. 


After this profile creation. Please cancel the job and restart it. We are still in the goto meeting https://global.gotomeeting.com/join/384318893 if you are available to join 

```

#### profile fix offload sqoop sql subpartitions null query
1) run sqld360 on sqlid 
2) get the standalone sql and format using http://dpriver.com/pp/sqlformat.htm
3) check this url (https://gist.github.com/karlarao/be752f09ac43f324f63e668607db12a4) on how to rewrite OR to UNION ALL. you would still need the binds from standalone sql. best way is to just replace the SQL in the standalone sql
4) and then follow the steps below
```

-- test the rewrite sql (from 2hours to 3.25mins) 
dwbs001p5(ac26646): set timing on
dwbs001p5(ac26646): @00fnpu38hz98x_rewrite.sql

no rows selected

Elapsed: 00:03:25.44

P_SQLID
-------------
22s34g2djar10


-- monitoring
TXT
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
5,09/11/18 13:39:11,pts/3,podclodmdb07.corp.intranet,oracle,AC26646,oracle,SQL*Plus,,sqlplus@podclodmdb07.corp.intranet (TNS V1-V3),AC26646,WAITING,,ACTIVE,1246,17109,  49550,1060347644,22s34g2djar10,SELECT data_object_id              file_id         rela,cell single block physical read,112


-- create AWR snapshot 
exec dbms_workload_repository.create_snapshot;
PL/SQL procedure successfully completed.


-- fix
cd /db_backup_denx3/p1/gluent/karl

dwbs001s1(sys): @create_sql_profile-goodbad.sql
Enter value for goodsql_id: 22s34g2djar10
Enter value for goodchild_no (0):  <HIT ENTER>
Enter value for badsql_id: 00fnpu38hz98x
Enter value for badchild_no (0): <HIT ENTER>
Enter value for profile_name (PROF_sqlid_planhash):  <HIT ENTER>
Enter value for category (DEFAULT):  <HIT ENTER>
Enter value for force_matching (FALSE): <HIT ENTER>
Enter value for plan_hash_value: <HIT ENTER>
SQL Profile PROF_00fnpu38hz98x_ created.


-- verify 
set serveroutput off 
@00fnpu38hz98x_beforerewrite.sql
@dplan 

Note
-----
   - SQL profile PROF_00fnpu38hz98x_ used for this statement
   - Warning: basic plan statistics not available. These are only collected when:
       * hint 'gather_plan_statistics' is used for the statement or
       * parameter 'statistics_level' is set to 'ALL', at session or system level

dwbs001s1(sys): select name, created from dba_sql_profiles order by 2 asc;

NAME                CREATED
---------------     ---------------------------------------------------------------------------
PROF_00fnpu38hz98x_ 11-SEP-18 03.26.46.000000 PM



1 row selected.

dwbs001s1(sys):
dwbs001s1(sys):
dwbs001s1(sys): @profile_hints.sql
Enter value for profile_name: PROF_00fnpu38hz98x_
HINT


```



#### sqoop benchmarking
link here [sqoop benchmark](https://gist.github.com/karlarao/856815e1cae7f5496dcb50359b7c8485)

#### sqoop benchmark monitoring
link here [sqoop benchmark monitoring](https://gist.github.com/karlarao/e8fc763b8a67ddc30d727209bd40fed4)

#### get precision sampling remaining time
* run sqlmon.sql check Read_GB vs sum(bytes) of dba_segments 
* Read_GB is also available in sql monitor report
```
below is 572GB of 3547GB so far 


dwops01p2(ac26646): select sum(bytes)/1024/1024/1024 from dba_segments where segment_name = 'PFMODS_PS_HOUR_SUMM_T_GLT' and owner = 'STAGE';

SUM(BYTES)/1024/1024/1024
-------------------------
                3547.2627


Global Information
------------------------------
 Status              :  EXECUTING
 Instance ID         :  1
 Session             :  GLUENT_ADM (8:33047)
 SQL ID              :  7wkzay47uuncm
 SQL Execution ID    :  16777216
 Execution Started   :  09/10/2018 12:39:42
 First Refresh Time  :  09/10/2018 12:39:48
 Last Refresh Time   :  09/12/2018 13:02:00
 Duration            :  174139s
 Module/Action       :  Gluent Offload Engine/Main
 Service             :  dwops01pu1
 Program             :  python@pddcdbadm01.corp.intranet (TNS V1-V3)

Global Stats
======================================================================================================
| Elapsed |   Cpu   |    IO    | Application | Cluster  |  Other   | Buffer | Read | Read  |  Cell   |
| Time(s) | Time(s) | Waits(s) |  Waits(s)   | Waits(s) | Waits(s) |  Gets  | Reqs | Bytes | Offload |
======================================================================================================
|  174140 |  173103 |       62 |        0.04 |     0.01 |      975 |    37M |   1M | 572GB |  17.36% |
======================================================================================================

SQL Plan Monitoring Details (Plan Hash Value=2906763176)
======================================================================================================================================================================================================
| Id   |            Operation             |           Name            |  Rows   | Cost |   Time    | Start  | Execs |   Rows   | Read | Read  |  Cell   | Mem | Activity |      Activity Detail      |
|      |                                  |                           | (Estim) |      | Active(s) | Active |       | (Actual) | Reqs | Bytes | Offload |     |   (%)    |        (# samples)        |
======================================================================================================================================================================================================
|    0 | SELECT STATEMENT                 |                           |         |      |           |        |     1 |          |      |       |         |     |          |                           |
|    1 |   SORT AGGREGATE                 |                           |       1 |      |    174135 |     +6 |     1 |        0 |      |       |         |     |    42.35 | Cpu (3737)                |
|    2 |    PARTITION RANGE ALL           |                           |    344K |  853 |    102027 |     +6 |     1 |       4G |      |       |         |     |          |                           |
|    3 |     VIEW                         |                           |    344K |  853 |    174134 |     +6 |    70 |       4G |      |       |         |     |    55.55 | Cpu (4902)                |
| -> 4 |      TABLE ACCESS STORAGE SAMPLE | PFMODS_PS_HOUR_SUMM_T_GLT |    344K |  853 |    174139 |     +6 |    70 |       4G |   1M | 572GB |  17.36% | 17M |     2.11 | Cpu (183)                 |
|      |                                  |                           |         |      |           |        |       |          |      |       |         |     |          | cell smart table scan (3) |
======================================================================================================================================================================================================

```

#### sqoop query performance bug
https://apache.googlesource.com/sqoop/+/refs/heads/trunk/src/java/org/apache/sqoop/manager/oracle/OraOopOracleQueries.java
http://mail-archives.apache.org/mod_mbox/sqoop-user/201507.mbox/%3CCAPCS4qog5cgj_rWKzHXPyMy3oEYgj5fCu9oVNbW5w8X0AOB6qg@mail.gmail.com%3E
https://groups.google.com/a/cloudera.org/forum/#!msg/sqoop-user/gz61875BaaY/0CM56l6cezYJ
https://github.com/apache/sqoop/blob/trunk/src/java/org/apache/sqoop/manager/oracle/OraOopOracleQueries.java
https://github.com/apache/sqoop/commits/trunk/src/java/org/apache/sqoop/manager/oracle/OraOopOracleQueries.java?author=jarcec
https://github.com/apache/sqoop/commits/trunk/src/java/org/apache/sqoop/manager/oracle/OraOopOracleQueries.java?author=generalpiston


#### predicate push down by adding a join condition on filter key
details here: https://gist.github.com/karlarao/aa83cf27c879edffa8b324b4407b389e


#### troubleshoot query rewrite
details here: https://gist.github.com/karlarao/8ea0658af3a20f78247f26172069a8c1


#### object OFFLOAD_BIN does not exist
```

, hybrid_rows AS (SELECT /*+ full */ COUNT(*) c FROM "STAGE_REFARCH_H"."MTN_VMR002DISTN_AP_GLT"
                                                                       *
ERROR at line 4:
ORA-06564: object OFFLOAD_BIN does not exist


dwprod38(ac26646):
dwprod38(ac26646):
dwprod38(ac26646): set role GLUENT_OFFLOAD_ROLE;

Role set.

dwprod38(ac26646):
dwprod38(ac26646): set timing on
dwprod38(ac26646): set serveroutput off
dwprod38(ac26646): @00009_sqld360_914483_c7knq43dms5pj_2a_3_standalone_sql-fullhint.sql

PL/SQL procedure successfully completed.

Elapsed: 00:00:00.02

PL/SQL procedure successfully completed.

```

## sql and command line cheatsheet

### gluent command line 

#### offload 

check [gluent_offload.md](https://gist.github.com/karlarao/d14a075e53a78117bc9df630c756ffba)

##### automated offload 

check [offload automated.md](https://gist.github.com/karlarao/753744cf417ae689417c9e5169784ffd)

##### how to know where data is split

check [how to know where data is split.md](https://gist.github.com/karlarao/cef97a75327aa095bc52df569c7ab316)



#### present 

check [gluent_present.md](https://gist.github.com/karlarao/ea59298e69fdf91faa379cd65346126f)

### command line 

#### check and restart metad
```
@podclodmdb07.corp.intranet[CCDW001S1]: /home/oracle > alias chkm2
chkm2='${grid_home}/bin/crsctl status resource gluent-metad.${GLUENT_DBNAME}'

@podclodmdb07.corp.intranet[CCDW001S1]: /home/oracle > chkm2
NAME=gluent-metad.CCDW001S
TYPE=cluster_resource
TARGET=ONLINE                , ONLINE
STATE=ONLINE on podclodmdb08, OFFLINE


Metad is a cluster resource, so you can start as root:

su -
/u01/app/12.1.0.2/grid/bin/crsctl start resource gluent-metad.CCDW001S
```

#### get errors on conn log
```
-- to troubleshoot for errors 
grep -B5 Exception conn*log

-- then on impala web ui query the following 
query_state = EXCEPTION
```

#### get memory from sqlmon files get high memory SQL

[get memory from sqlmon files, get high memory SQL](https://gist.github.com/karlarao/ca6e43df59b21130995b2e17d39849cb)


#### get duration from sqlmon files

```
find . | grep sqlmon > sqlmonout.txt
for i in `cat sqlmonout.txt | grep -v ".zip"`; do 
	echo $i " "`cat $i | egrep "</duration>"` | grep -v invalid | grep -v "sqlmonout.txt"
done 


OR THIS 

find . -type f -exec grep -H "</duration>" {} \;

```


#### rename benchmark files 
```
## rename files to g-<filename>
for i in `ls -1 000*sql`; do 
	echo $i
	mv $i g-$i
done 

## remove the g- in filename
for i in `ls -1 g-*sql`; do 
	mv $i `h=$i; echo ${h:2}`
done 
```

#### move zip and planx files

```
find . > files.txt 

for i in `cat files.txt | grep zip`
do
        ls $i
        mv $i .
done


for i in `cat files.txt | grep planx`
do
        ls $i
        mv $i .
done
```


### sqlplus 

#### show 
```


-- show partitions
rpt_tabpart rrsv2 master_nwx_monthly_glt %

-- show data in partition key 
-- change the column and table accordingly 
select to_char(META_END_EFF_TMSTMP,'YYYYMM') , count(*) 
from STAGE.FW_LU_QUAL_GLT group by to_char(META_END_EFF_TMSTMP,'YYYYMM') order by 1;


-- shows all offloaded tables so far
rpt_offload_objects_t % % 
rpt_offload_objects % % 

-- show offload date for offloaded tables
@rpt_gluent_cmds % %

-- show parallel on gluent objects
rpt_gluent_parallel % %

-- show rewrite
rpt_rewrites dwptora% %
rpt_rewrite <schema> aggtable 


-- show view
rpt_view 



```

#### 10053 trace
```
alter session set tracefile_identifier='MY_10053';
alter session set events '10053 trace name context forever';
select /* hard parse comment */ * from emp where ename = 'SCOTT';
alter session set events '10053 trace name context off';
```
https://jonathanlewis.wordpress.com/2010/04/30/10053-viewer/
```
-- investigate high level using the viewer 
https://jonathanlewis.wordpress.com/2010/04/30/10053-viewer/

-- chase the query block registry pattern the ones that says "FINAL" using this grep
grep -nE '^[A-Z]{2,}:|Registe' dwops01s1_ora_69874_MY_10053.trc | less
https://www.slideshare.net/MauroPagano3/chasing-the-optimizer-71564184
```


#### looping synonyms and synonym chain
```

-- check table 
select owner, object_name, object_type 
from dba_objects 
where object_name like 'LATIS_NETWORX_TAX_EVENT_DTL%'
and object_type = 'TABLE'
/

-- check synonym
select * from dba_synonyms
where table_name like 'LATIS_NETWORX_TAX_EVENT_DTL%';

-- create synonym command
-- create or replace synonym STAGE.TRAF_TRAFFIC_8XX_DETAIL for STAGE_H.TRAF_TRAFFIC_8XX_DETAIL_GLT;

-- check looping chain of synonyms 
--https://stackoverflow.com/questions/247090/how-to-debug-ora-01775-looping-chain-of-synonyms
--https://tamimdba.wordpress.com/2011/03/12/ora-01775-looping-chain-of-synonyms/
select owner, synonym_name, connect_by_iscycle CYCLE
from dba_synonyms
where connect_by_iscycle > 0
connect by nocycle prior table_name = synonym_name
and prior table_owner = owner
union
select 'PUBLIC', synonym_name, 1
from dba_synonyms
where owner = 'PUBLIC'
and table_name = synonym_name
and (table_name, table_owner) not in (select object_name, owner from dba_objects
where object_type != 'SYNONYM');

-- check synonym translation 
-- https://www.experts-exchange.com/articles/8397/Resolving-Oracle-Object-Names-and-Synonym-Chains.html
select gluent_app.synonym_resolve_name('LATIS_NME_TRANS_DTL') from dual;

--GLUENT_APP.SYNONYM_RESOLVE_NAME('LATIS_NME_TRANS_DTL')
--------------------------------------------------------------------------------------------------------------------------------------
--PUBLIC.LATIS_NME_TRANS_DTL ->PUBLIC.LATIS_NME_TRANS_DTL (--SYNONYM LOOP--)


```

#### kill session
* locking the account of user GLUENT on the database side will stop any of the sqoop jobs
* just killing the session would trigger the respawn of those sessions and then re-execute the failed jobs 

```
-- SQL to kill just the DB sessions 

 INST    MINS   SID USERNAME      PROG       SQL_ID         CHILD PLAN_HASH_VALUE      EXECS   AVG_ETIME EVENT                P1TEXT                       P1 P2TEXT                       P2 P3TEXT                    P3 SQL_TEXT                       OFF IO_SAVED_% machine                        OSUSER
----- ------- ----- ------------- ---------- ------------- ------ --------------- ---------- ----------- -------------------- -------------------- ---------- -------------------- ---------- -------------------- ---------- ------------------------------ --- ---------- ------------------------------ ----------
    8      50   729 AC26646       sqlplus@po c56kcvzp6usrs      2      2455563375          1    2,862.48 enq: PS - contention name|mode            1347616774 instance                      8 slave ID                          41 WITH   origin_rows AS (SELECT  No           0 podclodmdb03.corp.intranet     oracle
    8      50  1509 AC26646       oracle@pdd c56kcvzp6usrs      0      2455563375          0       14.17 external table read  filectx              1.4006E+14 file#                         0 size                   65536                                No           0 podclodmdb03.corp.intranet     oracle
    5      50   220 AC26646       oracle@pdd c56kcvzp6usrs      0      2455563375          0        5.11 external table read  filectx              1.3988E+14 file#                         0 size                   65536                                No           0 podclodmdb03.corp.intranet     oracle
    5      50   516 AC26646       oracle@pdd c56kcvzp6usrs      0      2455563375          0        5.11 external table read  filectx              1.4001E+14 file#                         0 size                   65536                                No           0 podclodmdb03.corp.intranet     oracle


select /* usercheck */ 'alter system kill session ''' || s.sid || ', ' || s.serial# || ', @' || s.inst_id || ''' immediate;'
from gv$session s
where s.sql_id = 'c56kcvzp6usrs';


'ALTERSYSTEMKILLSESSION'''||S.SID||','||S.SERIAL#||',@'||S.INST_ID||'''IMMEDIATE;'
--------------------------------------------------------------------------------------------------------------------------------------------------------------------
alter system kill session '220, 13343, @5' immediate;
alter system kill session '516, 2171, @5' immediate;
alter system kill session '729, 22989, @8' immediate;
alter system kill session '1509, 47147, @8' immediate;

```



#### disable query rewrite
```
LOGIC: DISABLE the <table_name>_GLT_AGG objects and KEEP the GLT_CNT_AGG PARALLEL 1 

-- disable rewrite 
 @gen_disable_rewrite % %
 -- filter with:  4 and hybrid_view not like '%CNT%' and hybrid_view like '%GLT_AGG%'
 -- and hit r to re-run sql 

execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('NETWRXDM_H.NWX_PERF_FPM_DATA_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('NETWRXDM_H.NWX_PFM_E2E_DAY_SUMM_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('NETWRXDM_H.NWX_PFM_PS_HOUR_SUMM_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('NETWRXDM_H.NWX_PFM_SLA_KD_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('NETWRXDM_H.NWX_PFM_SLA_KE_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('RRSV2_H.FDM_BUS_DETAIL_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('RRSV2_H.MASTER_271_MONTHLY_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('RRSV2_H.MASTER_272_MONTHLY_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('RRSV2_H.MASTER_CSO_MONTHLY_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('RRSV2_H.MASTER_NWX_MONTHLY_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('RRSV2_H.MASTER_PUC_MONTHLY_GLT_AGG' , REWRITE_MODE=>'DISABLED')
execute SYS.DBMS_ADVANCED_REWRITE.ALTER_REWRITE_EQUIVALENCE('RRSV2_H.SUMMARY_PUC_MONTHLY_GLT_AGG' , REWRITE_MODE=>'DISABLED')

-- then run this to check if it got disabled 
@rpt_rewrites % %

OWNER                          NAME                           REWRITE_MO
------------------------------ ------------------------------ ----------
NETWRXDM_H                     NWX_PERF_FPM_DATA_GLT_AGG      DISABLED
NETWRXDM_H                     NWX_PERF_FPM_DATA_GLT_CNT_AGG  GENERAL
NETWRXDM_H                     NWX_PFM_E2E_DAY_SUMM_GLT_AGG   DISABLED
NETWRXDM_H                     NWX_PFM_E2E_DAY_SUMM_GLT__64SV GENERAL
 
```

#### fix parallelism on gluent objects 

```
LOGIC: DISABLE the <table_name>_GLT_AGG objects and KEEP the GLT_CNT_AGG PARALLEL 1 

@gen_gluent_parallel.sql
-- '
-- Please note that the sql generated includes the
-- CNT hybrid external table, which should not be
-- set to parallel.  Until this can be coded, manually
-- looking a the views and removing the CNT hybrid
-- view from the output is the only way...
-- '
alter table STAGE_H.ENS_REVENUE_DETL_GLT_CNT__U1OD parallel (degree 4);
 --               48

alter table STAGE_H.ENS_REVENUE_DETL_GLT_AGG_EXT parallel (degree 4);
 --      342,085,736

alter table STAGE_H.ENS_REVENUE_DETL_GLT_EXT parallel (degree 4);
 --      684,171,471

alter table STAGE_H.LATIS_NME_HDR_GLT_CNT_AGG_EXT parallel (degree 8);
 --              127

alter table STAGE_H.LATIS_NME_HDR_GLT_AGG_EXT parallel (degree 8);
 --       55,462,842

alter table STAGE_H.LATIS_NME_HDR_GLT_EXT parallel (degree 8);
 --      110,925,683

alter table STAGE_H.RJF_JOURNAL_DETAIL_GLT_CN_NFIS parallel (degree 8);
 --               16

alter table STAGE_H.RJF_JOURNAL_DETAIL_GLT_AGG_EXT parallel (degree 8);
 --      203,885,798

alter table STAGE_H.RJF_JOURNAL_DETAIL_GLT_EXT parallel (degree 8);
 --      407,771,595

temp/tmp_gluent_parallel.sql


cat temp/tmp_gluent_parallel.sql | grep -v GLT_CNT_AGG | grep alter
alter table STAGE_H.ENS_REVENUE_DETL_GLT_CNT__U1OD parallel (degree 4);
alter table STAGE_H.ENS_REVENUE_DETL_GLT_AGG_EXT parallel (degree 4);
alter table STAGE_H.ENS_REVENUE_DETL_GLT_EXT parallel (degree 4);
alter table STAGE_H.LATIS_NME_HDR_GLT_AGG_EXT parallel (degree 8);
alter table STAGE_H.LATIS_NME_HDR_GLT_EXT parallel (degree 8);
alter table STAGE_H.RJF_JOURNAL_DETAIL_GLT_CN_NFIS parallel (degree 8);
alter table STAGE_H.RJF_JOURNAL_DETAIL_GLT_AGG_EXT parallel (degree 8);
alter table STAGE_H.RJF_JOURNAL_DETAIL_GLT_EXT parallel (degree 8);

dwbs001s1(sys): @rpt_gluent_parallel.sql

OFFLOADED_OWNER OFFLOADED_TABLE                OWNER             TABLE_NAME                     ORIG       HYB          NUM_ROWS
--------------- ------------------------------ ----------------- ------------------------------ ---------- ---------- ----------
STAGE           ASSIA_PORTS_GLT                STAGE_H           ASSIA_PORTS_GLT_CNT_AGG_EXT             1          1         32
STAGE           ASSIA_PORTS_GLT                STAGE_H           ASSIA_PORTS_GLT_AGG_EXT                 1          1   22499280
STAGE           ASSIA_PORTS_GLT                STAGE_H           ASSIA_PORTS_GLT_EXT                     1          1   44998560
STAGE           AT_VENDOR_DEMOGRAPHIC_GLT      STAGE_H           AT_VENDOR_DEMOGRAPHIC_GLT_4NF0          1          1         64
STAGE           AT_VENDOR_DEMOGRAPHIC_GLT      STAGE_H           AT_VENDOR_DEMOGRAPHIC_GLT_EXT           1          1  249804531
STAGE           ENS_REVENUE_DETL_GLT           STAGE_H           ENS_REVENUE_DETL_GLT_CNT__U1OD          4          4         48
STAGE           ENS_REVENUE_DETL_GLT           STAGE_H           ENS_REVENUE_DETL_GLT_AGG_EXT            4          4  342085736
STAGE           ENS_REVENUE_DETL_GLT           STAGE_H           ENS_REVENUE_DETL_GLT_EXT                4          4  684171471
STAGE           EPWF_PAYMENT_GLT               STAGE_H           EPWF_PAYMENT_GLT_CNT_AGG_EXT            1          1         80
STAGE           EPWF_PAYMENT_GLT               STAGE_H           EPWF_PAYMENT_GLT_AGG_EXT                1          1   16729350
STAGE           EPWF_PAYMENT_GLT               STAGE_H           EPWF_PAYMENT_GLT_EXT                    1          1   33458700
STAGE           LATIS_NME_HDR_GLT              STAGE_H           LATIS_NME_HDR_GLT_CNT_AGG_EXT           8          1        127
STAGE           LATIS_NME_HDR_GLT              STAGE_H           LATIS_NME_HDR_GLT_AGG_EXT               8          8   55462842
STAGE           LATIS_NME_HDR_GLT              STAGE_H           LATIS_NME_HDR_GLT_EXT                   8          8  110925683
STAGE           RJF_JOURNAL_DETAIL_GLT         STAGE_H           RJF_JOURNAL_DETAIL_GLT_CN_NFIS          8          8         16
STAGE           RJF_JOURNAL_DETAIL_GLT         STAGE_H           RJF_JOURNAL_DETAIL_GLT_AGG_EXT          8          8  203885798
STAGE           RJF_JOURNAL_DETAIL_GLT         STAGE_H           RJF_JOURNAL_DETAIL_GLT_EXT              8          8  407771595

17 rows selected.


```


### webgui 

#### query EXCEPTION SQLs on impala ui 
```
-- on impala web ui query the following 
query_state = EXCEPTION
```

#### show the plan tree and profile of an impala query 

```
# from impala queries url you'll see the following details
Query ID: cf4ca4ae671b1332:8321cf0100000000
User: gluent@EXAMPLE.INTRANET
Database: default
Coordinator: poldn001.test.intranet

# put the coordinator host and query id on the URL 
https://<coordinator host>:25000/query_plan?query_id=<query id>

# final
https://poldn001.test.intranet:25000/query_plan?query_id=cf4ca4ae671b1332:8321cf0100000000
```



## errors

### ORA 29913 odciexttablefetch callout
https://stackoverflow.com/questions/9066191/sqlplus-error-on-select-from-external-table-ora-29913-error-in-executing-odcie
```
* make sure grants are done
* make sure synonyms resolve to the objects
```

### reread 

### ORA 600

### memory limit exceeded 
https://gist.github.com/karlarao/ca6e43df59b21130995b2e17d39849cb#then-run-the-high-load-sql-then-it-errors-with-the-following

### view error grant on offlaod
```
Oracle sql: GRANT SELECT ON "CTLQWEST_H"."EIS_DNETS_V" TO CTLQWEST
WARNING:Unable to grant dependent view (ORA-01720: grant option does not exist for 'ENTP.CDL_POSTAL_ADDRESS_DIM')
Oracle sql: GRANT SELECT ON "CTLQWEST_H"."EIS_DNETS_V" TO DWBI_ETL
WARNING:Unable to grant dependent view (ORA-01720: grant option does not exist for 'ENTP.CDL_POSTAL_ADDRESS_DIM')
Oracle sql: GRANT SELECT ON "CTLQWEST_H"."EIS_DNETS_V" TO CTLQWEST_READER_ROLE
WARNING:Unable to grant dependent view (ORA-01720: grant option does not exist for 'ENTP.CDL_POSTAL_ADDRESS_DIM')
Oracle sql: GRANT SELECT ON "CTLQWEST_H"."EIS_DNETS_V" TO MRKT
WARNING:Unable to grant dependent view (ORA-01720: grant option does not exist for 'ENTP.CDL_POSTAL_ADDRESS_DIM')

```

### cognos not null on view

see link here [cognos not null on view](https://gist.github.com/karlarao/eee4a0a5ccd261610bb15bc21df53b7f)
the fix was:
I had to change our process to pull directory from the GLT table as it will only pull 13 months of data and we never re-run prior months reports. This kept the framework intact.


<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>

