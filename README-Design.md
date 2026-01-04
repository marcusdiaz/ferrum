I have been using DBT and I am inspired to create a new tool that is better and more robust than dbt.
 
The problem with dbt is that you create a model with a single SQL-defined logic built in. This inhibits having different update rules for a target table.
 
Instead, the new tool should decouple the table defintion from update rules, and allow a user to define tables - sources and target, with decoupled mappings from source (or multiple source with joins) to a target. Then, the tool should allow a user to build flows that combine sources, targets and mappings from source(s) to target to create a pipeline.
 
I want to write it in Rust. It should be similar to dbt in that it allows a user to setup a project-based configuration. It should use a database to store the business/transformation logic, table definitions and mappings. Then it should connect to another database to execute a flow/pipeline which could be from a source file into a target (or multiple targets), or from some table(s) into new table(s). Targets should have default mappings/update rules but be overridden by the mapping object being used. For example, a target table might have an updated_at date that is configured for a default, but which could be overridden from the source in a mapping rule.
 
It needs to have a frontend that allows a user to visually see and create table definitions or mapping rules as well as create flows which are sequences of steps which are also first class objects. Steps can be resusable in multiple flows.
 
Flows should have some kind of scheduler, either to run at a scheduled time or upon a file arrival. It should support local storage, S3 or files landing on an FTP server, configurable locations in the tool metadata.
 
Without code, write a document designing this new tool set. Include metadata table names and design the backend at a high level as well as the frontend at a high level. Include individual process/programs/libraries that will handle the various executable code that causes it to run.

