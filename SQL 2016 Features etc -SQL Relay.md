# Fun with SQL Server 2016
## Scott Klein
### SQL Relay 6/10/16

* look up channel9.msdn.com for SQL videos from  MS
SQL Polybase - NoSQL implementation? Use SSMS to query non-relational data

Looking at stretchdb, always encrypted and DDM, temporal tables

Stretch to Azure - possible idea for BI implementation? Need to look at BI partner and discuss with Rob. Could be a possible driver for 2016 adoption?

Look up temporal tables. Can automatically move data in and out of a table into history. Test for historical delivery data?

####Why use stretch?
Still appears as a table, so requires zero app changes. Doesn't affect the query plan, as the table remains. Need to think about latency though. Indexes persist.

Cost reduction at Rascal? Would Azure be cheaper? Can we rely on our internet?

Creates a PaaS instance on its own. ONLY works with an Azure Paas instance.

Azure Active Directory (AAD) - Branch AD to Azure, allowing AD auth to cloud features

When you backup it only backs up the 'hot' local data. Restore manages automatically.

How does this work if you restore to another server? Scott didn't know.

On creating the stretch you define what data will be stretched, like a partitioning function.

Thinking on, this is pretty much partitioning to the cloud.
