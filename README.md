# Postgres Background Worker

pg_background is an extension for Postgres 9.5.
Initially this extension was shared by Robert Haas in PostgreSQL community for demo purpose. Some modification has been done, by Vibhor Kumar, to keep it in sync with latest version of Postgres version >=9.5. Extra error handling and command results has been added.

I added compiling for Visual Studio 2022 and configurations for PostgreSQL 12.10 and 14.2.

This module allows user to arbitrary command in a background worker and gives capability to users to launch 

1. VACUUM in background
2. Autonomous transaction implementation better than dblink way.
3. Allows to perform task like CREATE INDEX CONCURRENTLY from a procedural language 

This module comes with following SQL APIs:

1. ***pg_background_launch*** : This API takes SQL command, which user wants to execute, and size of queue buffer. This function returns the process id of background worker.
2. ***pg_background_result*** : This API takes the process id as input parameter and returns the result of command executed throught the background worker.
3. ***pg_background_detach*** : This API takes the process id and detach the background process which is waiting for user to read its results.

## Installation steps (Linux)

1. Copy the source code from repository.
2. set pg_config binary location in PATH environment variable
3. Execute following command to install this module
```sql
make
make install
```
After installing module, please use following command install this extension on source and target database as given below:
```sql
  psql -h server.hostname.org -p 5444 -c "CREATE EXTENSION pg_background;" dbname
```


## Manual compiling in Windows (Visual studio)

1. Clone the repository.
2. Open pg_background.sln from `/windows/vs2022/` (or other version of Visual Studio).
3. In the properties dialog, go to *Configuration Properties -> C/C++ -> General*. In *Additional Include Directories*, pull down the arrow in the right of the textbox and choose “<Edit…>”. Now, by pasting paths or by folder selection, add the following folders inside your PostgreSQL install directory in this order:
```
include\server\port\win32_msvc
include\server\port\win32
include\server
include
```

4. You’ll also need to set the library path. Under *Linker -> General*, in *Additional Library Directories*:
 ```
  lib\
 ```
 While you’re there, set *Link Library Dependencies* to **No**.

5. Click “OK”. The error highlights on your extension file should go away when you return to the source file.
6. Build the solution.
7. See output DLL in `pg_background\windows\vs2022\x64\Release\pg_background.dll`.
8. Copy this DLL to PostgreSQL `/lib/` folder.
9. Put `pg_background.control` and `pg_background--1.0.sql` into PostgreSQL `/share/extension/` folder.
10. Create extension
    1) Connect to postgres as superuser with password used upon installation: `psql -U postgres`
    2) Run the following: `CREATE EXTENSION IF NOT EXISTS pg_background;`
    3) Run the following commands to test the extension:
```
CREATE TABLE t(id integer);

SELECT * FROM pg_background_result(pg_background_launch('INSERT INTO t SELECT 1')) AS (result TEXT);

SELECT * FROM t;
```

## Usage:

To execute a command in background user can use following SQL API
```sql
SELECT pg_background_launch('SQL COMMAND');
```

To fetch the result of command executed background worker, user can use following command:
```sql
SELECT pg_background_result(pid)
```

**pid is process id returned by pg_background_launch function**

## Example:

```sql
SELECT pg_background_launch('vacuum verbose public.sales');
 pg_background_launch 
----------------------
                11088
(1 row)


SELECT * FROM pg_background_result(11088) as (result text);
INFO:  vacuuming "public.sales"
INFO:  index "sales_pkey" now contains 0 row versions in 1 pages
DETAIL:  0 index row versions were removed.
0 index pages have been deleted, 0 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
INFO:  "sales": found 0 removable, 0 nonremovable row versions in 0 out of 0 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 0 unused item pointers.
Skipped 0 pages due to buffer pins.
0 pages are entirely empty.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
INFO:  vacuuming "pg_toast.pg_toast_1866942"
INFO:  index "pg_toast_1866942_index" now contains 0 row versions in 1 pages
DETAIL:  0 index row versions were removed.
0 index pages have been deleted, 0 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
INFO:  "pg_toast_1866942": found 0 removable, 0 nonremovable row versions in 0 out of 0 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 0 unused item pointers.
Skipped 0 pages due to buffer pins.
0 pages are entirely empty.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
 result    
--------
 VACUUM
(1 row)

```

If user wants to execute the command wait for result, then they can use following example:
```sql
SELECT * FROM pg_background_result(pg_background_launch('vacuum verbose public.sales')) as (result TEXT);
INFO:  vacuuming "public.sales"
INFO:  index "sales_pkey" now contains 0 row versions in 1 pages
DETAIL:  0 index row versions were removed.
0 index pages have been deleted, 0 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
INFO:  "sales": found 0 removable, 0 nonremovable row versions in 0 out of 0 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 0 unused item pointers.
Skipped 0 pages due to buffer pins.
0 pages are entirely empty.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
INFO:  vacuuming "pg_toast.pg_toast_1866942"
INFO:  index "pg_toast_1866942_index" now contains 0 row versions in 1 pages
DETAIL:  0 index row versions were removed.
0 index pages have been deleted, 0 are currently reusable.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
INFO:  "pg_toast_1866942": found 0 removable, 0 nonremovable row versions in 0 out of 0 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 0 unused item pointers.
Skipped 0 pages due to buffer pins.
0 pages are entirely empty.
CPU 0.00s/0.00u sec elapsed 0.00 sec.
 result 
--------
 VACUUM
(1 row)
```
