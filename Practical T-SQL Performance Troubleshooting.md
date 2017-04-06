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
