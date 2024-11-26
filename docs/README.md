# Setting Up QGIS User Profiles with PostgreSQL Role Management
> This guide explains how to create user profiles in QGIS that align with PostgreSQL role-based access control for managing permissions on spatial data.

## 1. Create Roles and Users in PostgreSQL
### Log into PostgreSQL
```bash
psql -U postgres -h localhost
```

### Create Group Roles
Define roles based on access needs (e.g., `viewer`, `editor`, `admin`).
```sql
CREATE ROLE viewer NOINHERIT;
CREATE ROLE editor NOINHERIT;
CREATE ROLE admin NOINHERIT;
```

### Create User Roles
> Assign each user to appropriate roles.

```sql
CREATE ROLE alice WITH LOGIN PASSWORD 'password_for_alice';
CREATE ROLE bob WITH LOGIN PASSWORD 'password_for_bob';

-- Assign roles
GRANT viewer TO alice;
GRANT editor TO bob;
```
### Grant Permissions on Database and Tables
```sql
GRANT CONNECT ON DATABASE spatial_db TO viewer, editor, admin;
GRANT USAGE ON SCHEMA public TO viewer, editor, admin;

-- Table-specific permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO viewer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO editor;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin;
```
## 2. Configure User Profiles in QGIS
### Create QGIS User Profiles
- Open QGIS.
- Go to `Settings > User Profiles > New Profile…`
- Name profiles according to the PostgreSQL roles, e.g., `ViewerProfile`, `EditorProfile`, `AdminProfile`.
- Each profile will have its own configuration folder.

### Configure Database Connections for Each Profile
- Switch to the specific `QGIS profile` via `Settings > User Profiles.`
- Open `Data Source Manager` via `Layer > Add Layer > Add PostGIS Layer…`
- Click `New` to create a connection for each profile.

### Set Up PostgreSQL Connection
> For each profile, configure the PostgreSQL connection to match the role credentials:

**Connection Details:**

- `Name`: PostgreSQL Viewer, PostgreSQL Editor, etc.
- `Host`: localhost or database server IP.
- `Port`: 5432
- `Database`: spatial_db
- `Username`: Specific PostgreSQL user (e.g., alice for viewer).
- `Password`: User’s password.


## 4. General Operation
### Create a Database
> Create a Database `human_resource` with user `postgres` and comment `This is human resource database`
```
	-- Database: human_resource
	-- DROP DATABASE IF EXISTS human_resource;
	CREATE DATABASE human_resource
		WITH
		OWNER = postgres
		ENCODING = 'UTF8'
		LOCALE_PROVIDER = 'libc'
		TABLESPACE = pg_default
		CONNECTION LIMIT = -1
		IS_TEMPLATE = False;
	COMMENT ON DATABASE human_resource
		IS 'This is human resource database';
```
### Create a Role
> Create role `hr_adm_grp`. No login and password is set and db creation is not allowed because it is a role
```
	-- Role: hr_adm_grp
	-- DROP ROLE IF EXISTS hr_adm_grp;
	CREATE ROLE hr_adm_grp WITH
	  NOLOGIN
	  NOSUPERUSER
	  INHERIT
	  NOCREATEDB
	  NOCREATEROLE
	  NOREPLICATION
	  NOBYPASSRLS;
```

### Create an User
> Create user `hr_adm` with password `1234`
```
	-- Role: hr_adm
	-- DROP ROLE IF EXISTS hr_adm;

	CREATE ROLE hr_adm WITH
	  LOGIN
	  NOSUPERUSER
	  INHERIT
	  NOCREATEDB
	  NOCREATEROLE
	  NOREPLICATION
	  NOBYPASSRLS
	  PASSWORD '1234';
```
### Manage permissions
> To manage permissions at the group level. A `group role` like `hr_adm_grp` might represent all HR administrators, and `individual users` like `hr_adm` can inherit the group's permissions.
```
GRANT hr_adm_grp to hr_adm
```

### GRANT Role to Database
> To allow the role `hr_adm_grp` to `connect`,`create schema`,`create temporary tables` in `human_resource` database.
```
GRANT ALL ON DATABASE human_resource to hr_adm_grp
```

### Create Read only `Group` and `User`
> This `Group` will all its users only to read access to the Database
```
	-- Role: hr_ro_grp
	-- DROP ROLE IF EXISTS hr_ro_grp;

	CREATE ROLE hr_ro_grp WITH
	  NOLOGIN
	  NOSUPERUSER
	  INHERIT
	  NOCREATEDB
	  NOCREATEROLE
	  NOREPLICATION
	  NOBYPASSRLS;

	COMMENT ON ROLE hr_ro_grp IS 'This is a read only group';
```

> `Users` will get the read access to the Database`
```
	-- Role: hr_ro
	-- DROP ROLE IF EXISTS hr_ro;

	CREATE ROLE hr_ro WITH
	  LOGIN
	  NOSUPERUSER
	  INHERIT
	  NOCREATEDB
	  NOCREATEROLE
	  NOREPLICATION
	  NOBYPASSRLS
	  PASSWORD '1234';

	COMMENT ON ROLE hr_ro IS 'This is a read only user';
```
> Manage `permissions`
```
GRANT hr_ro_grp TO hr_ro;
```
```
GRANT TEMPORARY, CONNECT ON DATABASE human_resource to hr_ro_grp;
```

### Default Previlages Setup
> `Table GRANT`. These permissions are `automatically granted` to roles when certain objects (Tables or Functions) are created.
```
ALTER DEFAULT PRIVILEGES FOR ROLE postgres
GRANT ALL ON TABLES TO hr_adm_grp;

ALTER DEFAULT PRIVILEGES FOR ROLE postgres
GRANT SELECT ON TABLES TO hr_ro_grp;
```

> `Function GRANT`.
```
ALTER DEFAULT PRIVILEGES FOR ROLE postgres
GRANT EXECUTE ON FUNCTIONS TO hr_adm_grp;
```

```
ALTER DEFAULT PRIVILEGES FOR ROLE postgres
GRANT EXECUTE ON FUNCTIONS TO hr_ro_grp;
```
> To Check the database `version` use `select version()`

> For current date use `select current_date`

### Delete a Role
> Suppose you want to delete the role `viewer`
 
1. View existing roles
In the psql command-line interface, you can list all roles with the `\du` command
```
\du
```

2. Check if `viewer` owns any objects
```
SELECT n.nspname AS schema_name,c.relname AS object_name,c.relkind AS object_type FROM pg_class c JOIN pg_roles r ON c.relowner = r.oid JOIN pg_namespace n ON c.relnamespace = n.oid WHERE r.rolname = 'viewer';
```

3. Reassign ownership of viewer's objects to admin
```
REASSIGN OWNED BY viewer TO admin;
```

4. Drop any remaining objects owned by viewer:
```
DROP OWNED BY viewer;
```

5. Revoke memberships if viewer is part of other roles:
```
REVOKE other_role FROM viewer;
```

6. Delete the viewer role:
```
DROP ROLE viewer;
```

## 5. psql
> Used by developers and administrators as it is easy to operate to and from a linux machine

### Connect to DB
1. Start from `start > SQL shell(psql)` and give credentials

`Server:` localhost
`Database:` postgres
`Port:` 5432
`Username:` postgres
`Password`: 1234

### View Databases
> All the database created can be viewed by the following command
```
\l
```
### Connect to a DB
> Connect to a specific Database
```
\c human_resource
```
### View Tables
> To view all the tables in the database
```
\dt
```
### List Tables and Views
> To view all the tables and views in the database
```
\d
```
### Describe Table Structure
> To check the field configuration of the attribute tables
```
\d hr_adm
```
### Connected SCHEMA's
To view the schema under which the table is located
```
\dn
```
### Connected Users
> To see the user who are maintaining the table
```
\du
```
### Connected Views
> To check which views are dependent on the table
```
\dv
```
### Connected Functions
> To check which functions are dependent on the table
```
\df
```
### Find timings
> Use it when to know the time consumed for query execution.
```
\timing
```
### Get help of psql
> This command generates a complete list of the help of psql
```
\?
```
### Ask for help in commands
> \h command generates the specific help related to the command
```
\h DROP TABLE
```
```
\h CREATE SEQUENCE
```
### Run saved .sql script
> \i allows a user to the execute a saved sql file from the psql
```
\i path of .sql File
```

### Create views
> A demo command for creating a view under public SCHEMA
```
CREATE OR REPLACE VIEW public.hr_emp_vw AS SELECT * FROM hr_emp;
```
### Delete a View
> This command deletes a table FROM the public SCHEMA
```
DROP VIEW IF EXISTS public.hr_emp_vw;
```

### Create Table
> This command generates a table under public SCHEMA with some following attributes.
```
	-- Table: public.hr_emp

	-- DROP TABLE IF EXISTS public.hr_emp;

	CREATE TABLE IF NOT EXISTS public.hr_emp
	(
		hr_emp_id bigint NOT NULL,
		hr_emp_name character varying(100) COLLATE pg_catalog."default",
		hr_sal bigint,
		CONSTRAINT hr_emp_pk PRIMARY KEY (hr_emp_id)
	)

	TABLESPACE pg_default;

	ALTER TABLE IF EXISTS public.hr_emp
		OWNER to hr_adm;

	REVOKE ALL ON TABLE public.hr_emp FROM hr_ro_grp;

	GRANT ALL ON TABLE public.hr_emp TO hr_adm;

	GRANT ALL ON TABLE public.hr_emp TO hr_adm_grp;

	GRANT SELECT ON TABLE public.hr_emp TO hr_ro_grp;
```

### Exit
```
\q
```

## 6. Table Management
### Create a simple DB

> Make a minimal Database with owner `postgres` and Table space `pg_default`
```
	-- Database: employee
	-- DROP DATABASE IF EXISTS employee;
	CREATE DATABASE employee
		WITH
		OWNER = postgres
		ENCODING = 'UTF8'
		LC_COLLATE = 'English_United States.1252'
		LC_CTYPE = 'English_United States.1252'
		LOCALE_PROVIDER = 'libc'
		TABLESPACE = pg_default
		CONNECTION LIMIT = -1
		IS_TEMPLATE = False;
```
### Create Tables
> 1. Create table with name `dept`, TABLESPACE `pg_default` owner `postgres`
```
	-- Table: public.dept
	-- DROP TABLE IF EXISTS public.dept;
	
	CREATE TABLE IF NOT EXISTS public.dept
	(
		dept_id bigint NOT NULL,
		dept_name character varying(100) COLLATE pg_catalog."default" NOT NULL,
		dept_role character varying(100) COLLATE pg_catalog."default",
		CONSTRAINT dept_pk PRIMARY KEY (dept_id)
	)

	TABLESPACE pg_default;

	ALTER TABLE IF EXISTS public.dept
		OWNER to postgres;
```
> 2. create another table `emp` with owner `postgres`, pk = `emp_pk`, fk = `dept_fk`
```
	-- Table: public.emp
	-- DROP TABLE IF EXISTS public.emp;
	CREATE TABLE IF NOT EXISTS public.emp
	(
		emp_id bigint NOT NULL,
		dept_id bigint,
		emp_name character varying(100) COLLATE pg_catalog."default" NOT NULL,
		emp_sal bigint,
		emp_role character varying(100) COLLATE pg_catalog."default",
		CONSTRAINT emp_pk PRIMARY KEY (emp_id),
		CONSTRAINT dept_fk FOREIGN KEY (dept_id)
			REFERENCES public.dept (dept_id) MATCH SIMPLE
			ON UPDATE CASCADE
			ON DELETE CASCADE
	)

	TABLESPACE pg_default;
	
	ALTER TABLE IF EXISTS public.emp
		OWNER to postgres;
```
### Insert records
> 3. INSERT records into `dept` table
```
INSERT INTO public.dept(dept_id, dept_name, dept_role) VALUES(10, 'CRM', 'Mr. Mizan');

INSERT INTO public.dept(dept_id, dept_name, dept_role) VALUES(20, 'SAP', 'Mr. Bart');

INSERT INTO public.dept(dept_id, dept_name, dept_role) VALUES(30, 'Delivery', 'Mr. Nelson');
```

> 4. Insert rows into `emp` table
```
INSERT INTO public.emp(emp_id, dept_id, emp_name, emp_sal, emp_role) VALUES(1, 10, 'Mr. Selim', 5000, 'DBA');

INSERT INTO public.emp(emp_id, dept_id, emp_name, emp_sal, emp_role) VALUES(2, 10, 'Mr. Jaman', 8000, 'DBA');

INSERT INTO public.emp(emp_id, dept_id, emp_name, emp_sal, emp_role) VALUES(3, 20, 'Mr. Nibir', 4000, 'Typist');

INSERT INTO public.emp(emp_id, dept_id, emp_name, emp_sal, emp_role) VALUES(4, 20, 'Mr. Imran', 6000, 'Typist');

INSERT INTO public.emp(emp_id, dept_id, emp_name, emp_sal, emp_role) VALUES(5, 30, 'Mr. Imam', 4000, 'Guard');

INSERT INTO public.emp(emp_id, dept_id, emp_name, emp_sal, emp_role) VALUES(6, 30, 'Mr. Tamim', 4000, 'Guard');
```
### Update records
> Update statement will CASCADE to effect of the updates to its all dependent.
```
UPDATE public.dept SET dept_id = 40 WHERE dept_id = 20;
```
### Import a Remote Database (Option 1)
?> Import complete SCHEMA procedure.

> We want to access `employee` or another Database information from `human_resource` Database. Open `query tool` from `human_resource` Database

> Step-1: Create extension to access the remote data
```
CREATE EXTENSION postgres_fdw SCHEMA public;
GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO Postgres;
```
> Step-2: Create Server and give access to postgres user.
```
CREATE SERVER employee_server FOREIGN DATA WRAPPER postgres_fdw OPTIONS(host 'localhost', dbname 'employee' port '5432');
ALTER SERVER employee_server OWNER TO postgres;
```
> Step-3: Create user mapping for postgres user
```
CREATE USER MAPPING FOR postgres SERVER employee_server OPTIONS (user 'postgres', password '1234');
```
> Step-4: Create SCHEMA `public_emp` and GRANT access `all` to user.
```
CREATE SCHEMA public_emp;
GRANT ALL ON SCHEMA public_emp TO postgres;
```
> Step-5(A): Import the `public` SCHEMA from `employee` Database to `public_emp` SCHEMA into `human_resource` database. It will import all tables from the schema.
```
IMPORT FOREIGN SCHEMA public FROM SERVER employee_server INTO public_emp;
```

!> Step-5(B): To import any specific table `dept` use the statement below.
```
IMPORT FOREIGN SCHEMA public LIMIT to (dept) FROM SERVER employee_server INTO public_emp;
```
> Step-6: Check the data imported
```
select * from public_emp.dept;
select * from public_emp.emp;
```

### Import a Remote Database (Option 2)
?> Create a FOREIGN Table which will point to the source database table. 
> 1. Step-1 to Step-3 will be the same.
> 2. Step-4: Create a `sales` table in `employee` Database
```
CREATE TABLE IF NOT EXISTS public.sales
-- Table: public.sales
-- DROP TABLE IF EXISTS public.sales;

CREATE TABLE IF NOT EXISTS public.sales
(
    sales_id bigint NOT NULL,
    sales_person character varying(100) COLLATE pg_catalog."default",
    sales_product character varying(100) COLLATE pg_catalog."default",
    CONSTRAINT sales_pkey PRIMARY KEY (sales_id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.sales OWNER to postgres;
```

> 3. Enter some data in `sales` table
```
INSERT INTO public.sales(sales_id, sales_person, sales_product)
VALUES (100,'Mr. Chang', 'Laptop');

INSERT INTO public.sales(sales_id, sales_person, sales_product)
VALUES (101,'Mr. Kang', 'Bag');

INSERT INTO public.sales(sales_id, sales_person, sales_product)
VALUES (102,'Mr. Lue', 'Memory Card');

select * from public.sales;
```

> 4. Step-5: Create a FOREIGN table `sales` in the target Database under public SCHEMA of `human_resource` Database
```
CREATE FOREIGN TABLE IF NOT EXISTS public.sales
(
    sales_id bigint NOT NULL,
    sales_person character varying(100) COLLATE pg_catalog."default",
    sales_product character varying(100) COLLATE pg_catalog."default"
)

SERVER employee_server
OPTIONS (schema_name 'public', table_name 'sales');

ALTER TABLE IF EXISTS public.sales OWNER to postgres;

SELECT * FROM sales;
```

### Import CSV data
> download from ./assets/dummy_companies.csv
from the [link](https://github.com/MizanUrp11/WEB-GIS-documentation/tree/master/docs)

1. Create a Database `organization` and Table `org` to import the CSV
```
	-- Database: organization
	-- DROP DATABASE IF EXISTS organization;

	CREATE DATABASE organization
		WITH
		OWNER = postgres
		ENCODING = 'UTF8'
		LC_COLLATE = 'English_United States.1252'
		LC_CTYPE = 'English_United States.1252'
		LOCALE_PROVIDER = 'libc'
		TABLESPACE = pg_default
		CONNECTION LIMIT = -1
		IS_TEMPLATE = False;
```

2. Generate sql configuration based on the CSV

> Start the sqlite3 command line tool
```cmd
sqlite3
```

> Create a temporary Database
```cmd
.open test.db
```

> Set mode to CSV
```cmd
.mode csv
```

> Import the CSV file
```
.import "C:/Users/.../assets/dummy_companies.csv"  dummy_companies
```

> Save sql file
```cmd
.output dummy_companies.sql
```

> Export the Table
```
.dump dummy_companies
```
> Clear the memory
```
.output stdout
```

> EXECUTE the sql in the query tool in public schema of organization 
```
CREATE TABLE IF NOT EXISTS "org"(
"org_index" TEXT, 
"organization_id" TEXT, 
"name" TEXT, "Website" TEXT,
"country" TEXT, "description" TEXT,
"founded" TEXT, "Industry" TEXT,
"number_of_employees" TEXT);
```
### Import and Export data in psql
> 4 A. By GUI: Import the data by GUI `org` > `Import/export data`
> 4 B. By psql
```Import
\copy public.org (org_index, organization_id, name, "Website", country, description, founded, "Industry", number_of_employees) FROM 'C:\Users\Mizan\Documents\works\documentation\docs\assets\dummy_companies.csv' DELIMITER ',' CSV HEADER;
```
```Export
\copy public.org (org_index, organization_id, name, "Website", country, description, founded, "Industry", number_of_employees) TO 'C:\Users\Mizan\Documents\works\documentation\docs\assets\dummy_companies_export.csv' DELIMITER ',' CSV HEADER;
```
> To Remove all data from the table `org` > `truncate`

### Create Table Views
?> views are virtual representations of data that show the outcomes of a SELECT query. Views can help to reduce the complexity of queries, and improve the query reusability.

!> It is recommended to always create separate `SCHEMA` from a single Source Table.

1. Create a new SCHEMA
```
CREATE SCHEMA app AUTHORIZATION postgres;
```
2. Create a view
```
CREATE VIEW app.org_vw AS
SELECT * FROM public.org;
ALTER TABLE app.org_vw OWNER TO postgres;
```

3. Update rows

!> Change in views will reflect in the original Database!
```
	UPDATE app.org_vw SET org_index=102 WHERE org_index=2;

	-- Check the data
	SELECT * FROM app.org_vw WHERE org_index=102;
	SELECT * FROM public.org WHERE org_index=102;

	-- Rollback
	UPDATE app.org_vw SET org_index='2' WHERE org_index='102';
```

4. Create views with filter `founded > '2000'`
```
	CREATE VIEW app.org_aftr_2000_vw
	WITH (
	  check_option=cascaded
	) AS
	SELECT * FROM public.org where founded > '2000';

	ALTER TABLE app.org_aftr_2000_vw OWNER TO postgres;
```

### Postgres Tyes
1. Create an new type `contact_typ` with `phone_no` and `email_id`.

```
create type public.contact_typ as
(
	phone_no bigint,
	email_id character varying(100)
);

alter type public.contact_typ owner to postgres;
```

2. Create `Enum` type
> These are options to insert automatically during data insert

```
create type public.shift_typ as enum
( 'day', 'night' );

alter type public.shift_typ owner to postgres;
```
3. Create `Range` type

```
CREATE TYPE public.shift_time_typ AS RANGE
(
    SUBTYPE=time
);

ALTER TYPE public.shift_time_typ
    OWNER TO postgres;
```

4. Create `emp` table again with references to the `types`

```
CREATE TABLE public.emp
(
    emp_id bigint NOT NULL,
    dept_id bigint NOT NULL,
    emp_name character varying(100) NOT NULL,
    emp_sal bigint,
    emp_role contact_typ,
    emp_shift shift_typ,
    emp_shift_time shift_time_typ,
    CONSTRAINT emp_pk PRIMARY KEY (emp_id),
    CONSTRAINT dept_fk FOREIGN KEY (dept_id)
        REFERENCES public.dept (dept_id) MATCH SIMPLE
        ON UPDATE CASCADE
        ON DELETE CASCADE
        NOT VALID
)
TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.emp OWNER to postgres;
```

5. INSERT data

```
INSERT INTO public.emp 
VALUES
(1, 10, 'Alice Johnson', 75000, ROW(1234567890, 'alice.johnson@example.com')::contact_typ, 'day', '[09:00,17:00)'::shift_time_typ);
```

### Postgres Functions
?> A function can validate data before insertion. Updating the function updates all processes that depend on it, reducing maintenance overhead. 

## Spatial Data Management
1. Import shp into PostgreSQL using PostGIS
or Import shp of DB manager from QGIS
2. Import non-spatial data using DB manager from QGIS

## SQL queries for non-spatial data
The examples below use the sample dataset - cities.csv

```
SELECT * FROM cities
SELECT * FROM cities LIMIT 10
SELECT name, country FROM cities
SELECT DISTINCT country FROM cities
SELECT COUNT(DISTINCT country) FROM cities
SELECT MAX(population) FROM cities
SELECT SUM(population) FROM cities
SELECT AVG(population) FROM cities
SELECT * FROM cities ORDER BY country
SELECT * FROM cities ORDER BY country ASC, population DESC
SELECT * FROM cities WHERE country='USA'
SELECT * FROM cities WHERE country='USA' OR country='CAN'
SELECT * FROM cities WHERE country='USA' AND population>1000000
SELECT * FROM cities WHERE country LIKE 'U%'
SELECT * FROM cities WHERE country LIKE '%A'
SELECT * FROM cities WHERE country LIKE '_S_'
SELECT * FROM cities WHERE country IN ('USA', 'CAN', 'CHN')
SELECT * FROM cities WHERE population BETWEEN 1000000 AND 10000000
```

## PostGIS

[Tutorial](tutorial.md)
