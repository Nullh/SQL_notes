# Practical T-SQL Performance Troubleshooting
## Klaus Aschenbrenner

* Performance monitoring Methodology
* Hardware & OS Configuration
* Performance Troubleshooting
* Database Maintenance

# Performance Monitoring Approaches
1. Establish a baseline
2. Identify bottlenecks
3. Make on change at a time <- This is important
4. Measure performance <- Compare change to baseline
5. Repeat as necessary

It's important to define when you have tuned enough. What is an acceptable level of tuning?

## Establishing a baseline
Run a profiler (or EE session) for 24 hours. Grab **SQL:BatchCompleted** and **RPC:Completed** filter on user database and application. Trace this to a file.
Load this into a table and create aggregates (SUM reads, writes, duration, CPU).
Capture 24 hours of perfmon (CPU, Batches/sec, Active Connections, Database IO). Use PAL to generate the trace.
Connect Excel and make graphs!

### Wait statistics
These track wait info at the instance level by time and reason.
We always have waits due to async resources and cooperative scheduling (enforced 4ms yields).
Good 'ol sys.dm_os_wait_stats gives you that info.

Queries exist in 3 states:
1. Running   <---|
2. Suspended     |
3. Runnable -----|

After 4ms any running query yields back to runnable, even of not completed.

Wait stats are cumulative since the last restart with no change metrics available.
Capture there stats with a timestamp and chuck them in a table.

### DEMO: Establishing a baseline
Generate a template from PAL to feed to perfmon.
Run the trace for 24h.
Benchmark Factory from Dell/Quest (30 day trial) -> define an RW workload for your DB to create a blocking workload. Specify how many users then allow a keying (emulated user wait time) or about .5 secs. So at 32 users we should have 64 transactions per second.
Let it run!
Check sys.dm_os_wait_stats
Run Paul Randall's analysis script to view relevant waits (or sp_blitz)
Check sys.dm_io_virtual_file_stats for file read contention. **Read stalls noted here! Send to Smiths**
Paul Randall has a script for this view too.
Run your perfmon blg through PAL.

## Hardware & OS Configuration
Bottleneck order:

Hardware
    OS
        SQL Server
            Tables/Indexes
                Queries

**Cool thing:** In perfmon you can look at any VM performance limits though VM Processor tlc. Possibly a good thing to check after migration/failover.

### CPU, Memory & Network Configuration
Check which layout you have, NUMA or SMP.
VMs are usually NUMA.
In an SMP system, we have multiple CPU cores and a single socket. The cores access memory through a single memory controller in the northbridge. This is a bottleneck!
NUMA (non uniform memory access) has multiple sockets and each socket forms a NUMA node. Each node has a seperate memory controller. The nodes are interconnected, so memory connected to one memory controller can access memory in another NUMA node using the interconnect. This is slower though, as it's physically further away! We can configure SQL Server to combat this. Memory in this system would be interleaved between nodes if NUMA is disabled in the bios. This means all memory access is the smae speed as the slowest access of the NUMA nodes. This sucks!
CoreInfo.exe (part of sysinternals) you can measure speed of memory access.

Power plan defaults to Balanced (blegh) to save power. This can throttle down CPUs or even park cores, which will kill performance.

Network cards are important too.
Jumbo frames up the MTU from 1518 bytes (allows about 1k of data) to about 9000 bytes. This means we can shift more data in a single round trip.
TCP chimney offload pushes overloaded work off to the CPU from the NIC.

### DEMO: CPU, Memory & Network
You can check if you're running in a NUMA node in task manager, or the SQL Server log at startup. Also DMV sys.dm_os_memory_nodes, but note that memory_node_id 64 is for the DAC and not a real node.
Check the power options in control panel. HOWEVER there may be BIOS level power plans configured. VMWare ESX default to balanced power plans as well!
Check the network card settings. If you have a gigabit connection, set it in Speed & Duplex.

### Storage Subsystems
Storage systems should be based on IOPS. OLTP workloads need lots of IOPS. More disks/controllers = more IOPS.
On a standard 15000 RPM disk, you're looking at ~200 IOPS. SSDs can handle thousands of IOPS.
RAID systems (hardware or software) provides better performance and/or fault tolerance.
RAID 5 requires 3 disks, writing parity information spread across disks. Because we are writing this parity info, we're using up IOPS, so RAID 5 is less great. Plus this allows only one disk to die. If there is a disk failure then we drop into degraded mode, and disk access and writing is very slow.
RAID 10 is a better choice as it increases fault tolerance, although it requires more disks.

Check in with msdb.dbo.suspect_pages to see if you have any potential checksum errors in your pages.
In case of a page corruption, take a tail log backup and restore only the damaged page from the last backup and your tail log.

### Testing Storage performance
We should be doing storage stress tests. Storage may be faulty or misconfigured. We need to look at MBps, IOPs and latency.
Microsoft offer Diskspeed and SQLIOSim to test your storage out. Also try Crystal DiskMark.
When testing a SAN you should test using a file larger than your SAN cache to get real disk figures.

## Prominent wait types
* PAGEIOLATCH -> Data not in buffer, SQL has to pull the page from storage.
* CXPACKET -> Parallelism waits.
* LCK_M_ -> Time taken to grab a lock
*

### PAGEIOLATCH
Waiting for SQL Server to read a page from disk. Indicates either a problem with the IO subsystem or we don't have enough memory to keep pages required in the buffer pool.
Check for a low PLE, large table or index scans, check the dm_io_virtual_file_stats and are you having CXPACKET waits too?

### Memory Models
**Conventional Memory Model**
Buffer pool can grow and shrink.

**Locked Pages**
You can use LPiM to force SQL to keep buffer pool in memory even if Windows needs to trim the working set for SQL Server. This can lead to an out of memory situation for the server.
If you're using this setting you need to set a limit on how much memory SQL Server can use. You can set this to the server max but leave either 2GB or 10%, unless you're running other apps.

**Large Pages**
SQL Server pages are 8k, but on 32-bit windows a page is 4k.
With a 64-bit server you can use larger pages, but this means SQL Server will need to grab all its memory when it starts.
It might be useful in certain circumstances, but in general not recommended.

### CXPACKET
ALL that this wait type tells you is that there are parallel execution plans, not whether they are good or bad!
A worker thread reports the CXPACKET wait whilst waiting for the slowest thread to complete, as well as the coordinator thread which reports this wait until the parallel process needs re-syncing.
You're best bet is checking sys.dm_os_waiting_tasks to see what threads are sat waiting and for what reason.
This wait type can also mean large table scans are happening.
Set MAXDOP to the number of cores in one NUMA node, this ensures a parallel plan stays in a single NUMA node.
Set cost threshold for parallelism to 30 and adjust as necessary.
Messing up these settings can lead to thread starvation, which will kill performance.
