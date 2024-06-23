---
title: In-place PostgreSQL upgrade
tags:
  - postgres
  - database
  - systemd
emoji: 🐘
queries:
  - in-place postgres upgrade
---

<Warning>

WIP: this is a **draft guide**. Make sure you know what you're doing before following this guide as **it's NOT a supported procedure**.

</Warning>


<Warning>

This procedure is **highly discouraged** as it may lead to serious data loss.

</Warning>

<Note>

Take advantage of the [zero downtime migration tool](https://docs.apiscp.com/admin/Migrations%20-%20server/#migration-components) to get those sites over that need the newer PostgreSQL release while evaluating the impact it can have.

</Note>

<Warning>

If you still chose to proceed with an in-place migration, **make sure your data is backed up before proceeding.**

</Warning>

## Checklist
- [ ] Set maintenance window
- [ ] upcp -sb 1 hour before maintenance, to make sure everything works
- [ ] Do all steps in exact order
- [ ] Take notes on behaviour


## Procedure

```bash
export PG_FROM_VERSION="14"
export PG_TO_VERSION="16"
export PG_UPGRADE_TIME=$(date +%F-%T)
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"

# Perform a backup of all databases
pg_dumpall > "/home/pg-all-$PG_UPGRADE_TIME.sql"

# Temporarily disable Monit
systemctl stop monit.service apiscp.service

# Set new Postgres version
cpcmd scope:set cp.bootstrapper pgsql_version "$PG_TO_VERSION"

# Remove pgdg repo to fetch fresh one
mv /etc/yum.repos.d/pgdg-redhat-all.repo{,.old}
mv /etc/yum.repos.d/timescaledb.repo{,.old}

# Install new packages
upcp -sbf packages/install

upcp -sbf pgsql/install
systemctl stop postgresql-$PG_FROM_VERSION
systemctl stop postgresql-$PG_TO_VERSION
systemctl reset-failed
upcp -sbf pgsql/install

# Login as Postgres to initdb and upgrade
su postgres

# Initialize a new DB
/usr/pgsql-$PG_TO_VERSION/bin/initdb -D /var/lib/pgsql/$PG_TO_VERSION/data

# Add timescale to shared libs, otherwise install will fail
echo "shared_preload_libraries = 'timescaledb'" >> /var/lib/pgsql/$PG_TO_VERSION/data/postgresql.conf

# Reset WAL
/usr/pgsql-16/bin/pg_resetwal -f /var/lib/pgsql/16/data/

# Perform the upgrade with the command that fits your OS version
# RHEL <8
/usr/pgsql-$PG_TO_VERSION/bin/pg_upgrade -b /var/lib/pgsql/$PG_FROM_VERSION/bin -B /var/lib/pgsql/$PG_TO_VERSION/bin -d /var/lib/pgsql/$PG_FROM_VERSION/data -D /var/lib/pgsql/$PG_TO_VERSION/data -j 4
# RHEL 8+
/usr/pgsql-$PG_TO_VERSION/bin/pg_upgrade -b /usr/pgsql-$PG_FROM_VERSION/bin -B /usr/pgsql-$PG_TO_VERSION/bin -d /var/lib/pgsql/$PG_FROM_VERSION/data -D /var/lib/pgsql/$PG_TO_VERSION/data -j 4

exit

# Start new Postgres
systemctl start postgresql-$PG_TO_VERSION

# Start Monit service again
systemctl start monit.service apiscp.service
```

<Warning>

In case anything goes wrong, **after switching back version and reinstalling packages**, you can try restoring the databases with the following command: `psql --clean -f "/home/pg-all-$PG_UPGRADE_TIME.sql" postgres`

</Warning>
