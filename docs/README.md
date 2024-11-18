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
[Tutorial](tutorial.md)