# Data Warehouses in Azure
## Simon Whiteley - Adatis

Free trial for Azure SQLDW at the mo.

### SPP vs MPP
SMP is symmetric multi processing (a standard SQL Server) CPU and storage are clustered, so different CPUs contend on the same storage unit.
MPP is massively parallel processing - Each compute node has its own storage. Separation of data to CPU. Dangerous because requires specialist design.

### Azure SQL DataWarehouse
Cloud implementation of MSs full PDW/APS stack.
In the physical version scaling is Added as quarter racks, which are basically 8 sets of database buckets.
This approach doesn't work for fast scaling in the cloud.

In azure you always have 60 distribution nodes (so data in a table is split 60 times). Because of this you need to be looking at terabytes of data to make this worthwhile.

### Scaling
So how do we scale with a fixed 60 distributions?
We change the amount of compute (DWU - Warehouse version of DTUs) where 100 DWUs are the same as a 1/4 rack (1 node)
So you add another 8 readers for every 100 DWUs, 8 readers per compute node
 So if we have 1 compute node (8 readers) how does it work with 60 distro nodes?
 One does it all. If you add more it splits distribution nodes to each compute node equally

 *NOTE Changing the scaling basically restarts the server*

 ### Data Import
 You can use BCP or SSIS to do this, but it's kind of a push & pull system, so it all goes through the control node, which is acting as a bottleneck.

 #### PolyBase
 A better method is to have each compute node pull in data. This is done using **PolyBase** (built into 2016) using HDFS the Hadoop file system.
 Blob storage is HDFS, chunked to 512 kb chunks. Each file is fragmented over different disk. It's dead fast.
 There's problems (look it up). Also if your flat files are compressed, it will not treat the split files, so it will be slow. Instead chunk the compressed files into 512k blocks.

 It looks like an SQL table in SSMS, accessing the table spins up the compute nodes and pulls the data from the HDFS files.

 ### Distributions
 How do I decide which data goes into which distribution? You want it to be levelled per distribution node.

 You can do:
 * Round Robin
 * HASH Column (distribute each unique value for a column in an individual distribution)
 * Replicated (copy it everywhere! It's not available in Azure yet)

 HASH is most common, but you need to pick the right distribution key to spread things evenly among distribution nodes.

 ### Execution plans

 There's the SQL database service, and a separate service that moves data around distinct from SQL.
 Because of this there's no plan caching because the DMS (data movement service) could be doing anything at query time.
 You CAN see pseudo execution plan using the EXPLAIN command
 The tuning workflow is write the query, run EXPLAIN <query> and see what it will do. You tune on the XML from EXPLAIN

 ### CTAS

 Create Table As Select - a syntax for inserting data into DW
 Kinda how it sounds. It's the fastest way to get data into a table. It's a non-logged bulk insert. It's pretty much the only decent way to get data into tables.
 This way there are not many tables. Instead we have lots of SPs with the pattern DROP TABLE -> CREATE TABLE AS SELECT (with OPTION LABEL just to make it easier to find in the DMVs) -> return the rowcount
 There's no @@ROWCOUNT in ASDW because of how it works. For now you need to write utility procs for it. There's lots of pre-written scripts around

### Resurce Classes

Resource classes are like resource governor. At 100 DWUs you have 4 concurrency slots, so you can run 4 queries. (DW is great for parallelism but bad at concurrency). Max concurrent queries scales up with DWUs up to 32. Once you scale up after that you can allocate more than one concurrency slot to a query.

You have small, medium, large and extra large resource classes which scale up per 100 DWUs. Small is the default! If you don't assign users to a larger resource class you might not be using all the capabilities of the system.
This how you manage concurrency for users.

### Partitioned Data Loads
In a DW the fastest way to load into a fact table is to insert a partitioned plus changed rows into a CTAS, switch out the old partition and switch in your table as the old partition.

### ETL vs ELT
ETL is the ol' SSIS version, so the work is done in SSIS then throw it through the control node. WHICH IS SLOW.
Instead we can just get the data in as fast as possible then do the transformation in DW.

### Statistics
ASDW does not auto create statistics!
Microsoft docs has pre-created utilities to create stats, but bear in mind you need to do this manually and it can be slow! Think about what stats you actually need.

### Surrogate keys

There's no identity columns in ASDW. You need to create them manually.
