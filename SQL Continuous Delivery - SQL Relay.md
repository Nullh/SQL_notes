# Continuous Delivery
## Gavin Campbell
### SQL Relay 6/10/2016

SQL Server Data Tools -> Look this up
Allows automated deployment, creates a development experience

An SSDT proj exists as a VS project, which can be change controlled. Database is defined as a series of scripts. But just bundling scripts presents problems if an object exists.

SSDT does this by creating an in-memory model of the database without SQL Server.

So how do we deploy? Run the scipts, but in a different way. It builds the project first then deploys it to localdb via creating a dacpac (a zip with xml).

This is a bit of a waste of time. Instead we can publish to an actual SQL instance. Pretty scary.

Instead we can add the dacpac to our build chain. On deployment it will compare the dacpac model to the deployed one and affect any changes.

#### Testing
How do we test this?
We bundle SQL scripts into a project with a .net wrapper. The unit tests then run against a deployed DB, you can create rules for the unit tests there.

So TeamServer can make sure any DBA can check the SQL before it is merged into master.
