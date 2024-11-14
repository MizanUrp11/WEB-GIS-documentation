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



[Tutorial](tutorial.md)