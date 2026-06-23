---
title: "pgclonedb"
description: "End-user guide for the pgclonedb PostgreSQL extension"
type: "docs"
weight: 9 
---

<img class="floatimg" src="/postgres/pgclonedb.png" alt="elephant" width="400"/>
<br><br>
<p>
This guide is written for users of the pgclonedb extension. 

Installation, role onboarding, and full API documentation are in the
project README: [https://github.com/raphideb/pgclonedb/blob/main/README.md](https://github.com/raphideb/pgclonedb/blob/main/README.md)
</p>
<div style="clear: both;"></div>

## What pgclonedb does

The extension exposes a small, safe set of database administration tasks
through the schema `pgclonedb`. Cloning a database, opening or closing it for
template use, managing schemas inside it, and (when enabled by the superuser)
managing roles and passwords are all available via
`SELECT pgclonedb.<function>(...)` calls.

Access is enforced by a naming convention. When the connected role is, for
example, `myapp_db_admin`, only databases named `myapp` or starting with
`myapp_` (such as `myapp_dev`, `myapp_v2`, `myapp_cicd_test`) are in scope.
Any attempt to reach databases of a different application is rejected with an
out-of-scope error.

## Getting started

The first step is to connect to the cluster as the application admin role
provided by the superuser. The connect string, role name, password are normally 
handed over during onboarding and follow the pattern `postgresql://<app>_db_admin:<password>@<host>:<port>/postgres`

A quick way to verify access and to see which functions are currently
available is the help function:

```sql
SELECT pgclonedb.help();
```

The output lists every function, its parameters, a usage example, and whether
the function is currently `enabled` or `DISABLED`. It also reminds of the
basic cloning workflow and the scoping rule, and is the recommended starting
point whenever something is unclear or a function does not behave as expected.

## The basic workflow: prepare and clone a database

The template database is the master copy from which clones are produced. By
convention it is kept **closed** at all times — locked in template mode, with
no connections allowed — so that it stays consistent and ready to be cloned
on demand. Applications never connect to the template directly; they connect
to clones.

The template is only opened temporarily by the application admin when its
contents need to change: loading initial data, applying schema updates,
creating application roles, importing reference data, and so on. As soon as
those changes are complete, the template is closed again. Outside of these
short maintenance windows, the template remains closed.

This leads to two distinct workflows:

- Preparing or updating the template — occasional, performed only by the
  application admin
- Cloning from the template — performed whenever a fresh working copy is
  needed

### Preparing or updating the template

When the template is first set up, or when its contents need to change later,
it is opened briefly, edited, and closed again:

```sql
-- 1. Unlock the template for editing
SELECT pgclonedb.open_database('myapp_source');

-- 2. Connect to 'myapp_source' and apply the changes
--    (load data, create schemas, create application roles, run migrations, ...)

-- 3. Lock the template back into clonable state
SELECT pgclonedb.close_database('myapp_source');
```

After step 3, the template is ready to be cloned at any time — and it should
remain in this state until the next planned change. Leaving the template
open between maintenance windows is discouraged: applications could
accidentally connect to it, ad-hoc changes could drift the master copy out
of its known-good state, and any active session would block subsequent clone
operations.

By default, `close_database` terminates active connections to the template.
When this is not desired (for example to be sure that no maintenance work is
still in progress), the parameter `terminate_backends := false` can be
passed — the call then fails instead of terminating sessions.

### Cloning from the template

Once the template is in its final, closed state, fresh clones can be produced
on demand with a single call:

```sql
SELECT pgclonedb.create_database(
    dbname   := 'myapp_dev',
    template := 'myapp_source'
);
```

Optional parameters are available for `owner`, `encoding`, `lc_collate`, and
`lc_ctype`. When `owner` is not specified, the new database inherits the
owner from the template.

*Note*: locales can only be changed when cloning from template0 or template1.

Before cloning begins, the extension checks whether there is enough free disk
space for the copy. By default, at least 10 GB must remain free after cloning.
This safety margin is managed by a superuser through the `configure` function:

```sql
-- Use a 20 GB margin instead of the default 10 GB
SELECT pgclonedb.configure(min_free_bytes := 21474836480);

-- Skip the margin, only check that the template fits
SELECT pgclonedb.configure(min_free_bytes := 0);

-- Disable the disk space check entirely
SELECT pgclonedb.configure(min_free_bytes := -1);

-- View current configuration
SELECT * FROM pgclonedb.show_config();
```

No `open_database` / `close_database` calls are needed around the clone
operation — the template is already in the correct state and stays that way
after cloning. The clone itself is opened automatically and is ready for
applications to connect to.

### Recovering an accidentally opened template

When the template was opened (for example for a quick inspection) and now
allows connections, the way back into the standard state is simply to close
it again:

```sql
SELECT pgclonedb.close_database('myapp_source');
```

## Removing a clone

Clones that are no longer needed can be dropped:

```sql
SELECT pgclonedb.drop_database('myapp_dev', force := true);
```

The `force` flag terminates any remaining connections to the target database
before dropping it. It defaults to `true`, so `SELECT
pgclonedb.drop_database('myapp_dev');` already terminates connections. Pass
`force := false` to make the call fail instead when active sessions are still
present.

To drop a Source/template database that was closed with `pgclonedb.close_database`, open it first with `pgclonedb.open_database`.

## Working with schemas

When a cloned database needs additional schemas — for example to separate
test data, analytics, or per-feature staging areas — the schema functions can
be used. Because schemas live inside a database, the database name needs to be
passed as well.

### Creating a schema

```sql
SELECT pgclonedb.create_schema(
    dbname      := 'myapp_dev',
    schema_name := 'app_data',
    owner       := 'myapp_app'   -- optional
);
```

The `owner` parameter is optional and defaults to the calling role.

### Dropping a schema

```sql
SELECT pgclonedb.drop_schema(
    dbname      := 'myapp_dev',
    schema_name := 'app_data',
    cascade     := true
);
```

Without `cascade`, the call fails when the schema still contains objects.
The system schemas `pg_catalog`, `information_schema`, and `public` are
protected and cannot be dropped.

## Working with roles

Role management is useful for creating dedicated application accounts (for
example a read-only reporting role) inside the cluster. Role names are not
restricted by the `<app>_` prefix — any name is allowed — but roles are
shared across the entire cluster, so the chosen name needs to be unique and
reasonably descriptive.

### Creating a role

```sql
SELECT pgclonedb.create_role(
    rolename   := 'app_readonly',
    password    := 'S3cure!',
    login       := true,             -- default
    createdb    := false,            -- default (true is rejected)
    createrole  := false,            -- default (true is rejected)
    conn_limit  := -1,               -- default (unlimited)
    valid_until := '2026-12-31'::timestamptz   -- optional
);
```

Superuser roles cannot be created through this interface, and roles ending in
`_db_admin` are reserved for the superuser to create. The `createdb` and
`createrole` attributes cannot be enabled here either - passing
`createdb := true` or `createrole := true` raises an error. If a role really
needs one of those privileges, ask the cluster superuser to grant it.

Note that roles are **cluster-wide**: a role you create is visible to, and (when
the role functions are enabled) manageable by, every other application admin on
the cluster. Choose descriptive, collision-free names and treat role management
as a shared-cluster operation.

### Resetting a password

When a password has been forgotten or needs rotating:

```sql
SELECT pgclonedb.reset_password(
    rolename    := 'app_readonly',
    new_password := 'N3wS3cure!',
    valid_until  := '2027-06-30'::timestamptz   -- optional
);
```

Passwords of superuser roles, and of other applications' `*_db_admin` roles,
cannot be reset through this interface.

A note on password handling: the new password is sent to the server in an
`ALTER ROLE ... PASSWORD '...'` statement. If the cluster is configured to log
DDL or modifying statements (`log_statement = 'ddl'` or `'mod'`), the plaintext
password can end up in the server log. If this matters for your environment,
ask the cluster superuser how password operations are logged before using this
function.

### Granting and revoking role membership

To compose permissions, one role can be granted into another:

```sql
SELECT pgclonedb.grant_role('app_readonly', 'app_writer');
SELECT pgclonedb.revoke_role('app_readonly', 'app_writer');
```

The role being granted or revoked cannot be a privileged or reserved role:
PostgreSQL predefined roles (`pg_*`), roles with `SUPERUSER`, `CREATEROLE`,
`CREATEDB`, or `BYPASSRLS`, the `pgclonedb_user` group, and any `*_db_admin`
role are all blocked by design. This applies equally to `grant_role` and
`revoke_role`.

### Dropping a role

```sql
SELECT pgclonedb.drop_role('app_readonly');
```

System roles (`postgres`, `pgclonedb_user`), other applications'
`*_db_admin` roles, and superuser roles are protected and cannot be dropped.

## Reviewing the audit log

Every operation performed through the extension is recorded. Each application
admin can review the entries that belong to their own role:

```sql
SELECT * FROM pgclonedb.get_audit_log(max_rows := 20);
```

Filtering by operation is also possible:

```sql
SELECT * FROM pgclonedb.get_audit_log(filter_operation := 'create_database');
```

The audit log is a useful first stop when verifying that a previous operation
succeeded, when reconstructing what happened during a specific time window, or
when preparing a question for the cluster superuser.

## Quick reference: pgclonedb.help()

The most compact way to see what is currently available, including the
enabled/disabled status of each function, is the `help` function:

```sql
SELECT pgclonedb.help();
```

The returned text lists all functions with their parameters and an example
call, the basic cloning workflow, and the scope rule. Each function name is
annotated with `[enabled]` or `[DISABLED]` based on the current cluster
configuration. This is the recommended starting point whenever a function
does not behave as expected, or when checking which operations are currently
allowed.

## When something does not work

When a function call fails with an error such as `function is currently
disabled` or `out of scope`, the reasons are typically one of the following:

- The function is disabled by default in this cluster (for example
  `create_role`, `drop_role`, `reset_password`, `grant_role`, `revoke_role`,
  `create_schema`, `drop_schema`) and needs to be enabled by the superuser
  before use.
- The targeted database, template, or schema is not within the scope of the
  connected `<app>_db_admin` role.
- The operation touches a protected object — a system database, a system
  role, another application's admin role, or a superuser role.
- The peer-authentication setup that the extension relies on internally is
  not available on the cluster.

In all of these cases, **the cluster superuser is the right contact**.
Reaching out with the exact error message, the SQL statement that produced
it, and (when relevant) a recent excerpt from `pgclonedb.get_audit_log()`
makes diagnosis fastest.

## End-to-end cheat sheet

A minimal example covering a one-off template preparation, a clone produced
later from the (already closed) template, a quick activity check, and the
clean-up of the clone:

```sql
-- one-off: prepare or update the template
SELECT pgclonedb.open_database('myapp_source');
-- ... connect to 'myapp_source' and load data, create schemas, create roles, etc. ...
SELECT pgclonedb.close_database('myapp_source');

-- any time afterwards: produce a clone from the closed template
SELECT pgclonedb.create_database('myapp_dev', 'myapp_source');

-- inspect recent activity
SELECT * FROM pgclonedb.get_audit_log(max_rows := 5);

-- clean up the clone when no longer needed
SELECT pgclonedb.drop_database('myapp_dev');
```
