
# Database Migration from Oracle 12c to PostgreSQL 

This documentation provides a step-by-step guide for migrating a database from Oracle to PostgreSQL using the `ora2pg` tool. It also covers the setup of asynchronous replication between three sites using PostgreSQL streaming replication. Additionally, it includes instructions for modifying the dependencies and application properties of a Grails application to work with the PostgreSQL database.

Please note that this documentation assumes a basic understanding of Oracle, PostgreSQL, and Grails. It is recommended to refer to the official documentation of each tool for more detailed information.

## Table of Contents

1. Pre-Migration Steps
    - Installing and Configuring PostgreSQL
    - Installing and Configuring `ora2pg`
    - Setting up Asynchronous Replication
    - Data Type compatibility Matrix (YET TO COMPLETE)

2. Database Migration Steps
    - Schema Creation
    - Rewriting of Functions, Procedures and Views
    - Database Level testing of Functions and Procedures (YET TO COMPLETE)
    - Data Migration

3. Grails Application Configuration
    - Modifying the DB date fetching function (YET TO COMPLETE)
    - Modifying build.gradle
    - Modifying application.yml
    - Modifying the kDemoService where the DB functions/procs are being called

4. Testing the Web Application
    - Pilot testing of functionalities (Attach the list, YET TO COMPLETE)
    - Testing CRUD operation
    - Testing 

5. Setting up Asynchronous Replication between Master and two Replica databases
    - Setting up the Master Database Server
    - Setting up the Replica database Server
    - Testing the replication status 

---

## 1. Pre-Migration Steps

### Installing and Configuring PostgreSQL

1. Install PostgreSQL on the target system by following the official installation guide for your operating system.

2. Configure PostgreSQL by modifying the `postgresql.conf` file. Adjust the following settings as per your requirements:
    - `listen_addresses`: Set it to the IP address of the server.
    - `max_connections`: Configure the maximum number of allowed connections.
    - `shared_buffers`: Set an appropriate value for system memory.
    - `wal_level`: Set it to `replica` for streaming replication.
    - `max_wal_senders`: Set the maximum number of WAL sender processes.
    - `archive_mode` and `archive_command`: Configure archiving settings if required.

3. Restart the PostgreSQL service to apply the configuration changes.

### Installing and Configuring `ora2pg`

1. Install `ora2pg` on the migration server by following the installation instructions provided in the `ora2pg` documentation.

2. Configure `ora2pg` by creating a configuration file (`ora2pg.conf`). Set the necessary Oracle and PostgreSQL connection details, schema mapping, and migration options. Refer to the `ora2pg` documentation for detailed configuration instructions.

### Setting up Asynchronous Replication

1. Prepare three PostgreSQL servers for replication: primary, standby1, and standby2. Follow the official PostgreSQL documentation for setting up streaming replication.

2. Configure the `postgresql.conf` file on the primary server:
    - Set `wal_level` to `replica`.
    - Set `max_wal_senders` to a value that allows the desired number of standby servers.
    - Set `wal_keep_segments` to an appropriate value.

3. Create a replication user and grant the necessary permissions on the primary server:
    ```sql
    CREATE USER replication_user REPLICATION LOGIN ENCRYPTED PASSWORD 'your_password';
    GRANT REPLICATION SLAVE ON *.* TO replication_user;
    ```

4. Configure the `recovery.conf` file on the standby1 and standby2 servers:
    - Set `standby_mode` to `on`.
    - Set `primary_conninfo` to specify the connection details of the primary server.
    - Set `restore_command` to specify the command for retrieving archived WAL files if using WAL archiving.

5. Start the PostgreSQL service on the standby servers.

6. Verify the replication setup by checking the logs and confirming that the replication process is running.

---

## 2. Database Migration Steps

### Schema Creation

1. Connect to the Oracle database using `ora2pg`:
   ```shell
   ora2pg --project_base your_project_directory --project_name your_project_name --init_project
   ```

2. Edit the generated

 `your_project_directory/your_project_name/config/ora2pg.conf` file:
   - Set the `ORACLE_DSN` parameter to the Oracle connection string.
   - Set the `SCHEMA` parameter to the Oracle schema you want to migrate.
   - Configure the `EXPORT_SCHEMA` and `EXPORT_TABLE` parameters to control the objects to be exported.

3. Generate the PostgreSQL schema creation script:
   ```shell
   ora2pg --project_base your_project_directory --project_name your_project_name --gen_schema
   ```

4. Verify the generated schema creation script (`your_project_directory/your_project_name/schema.sql`). Review it for any necessary modifications.

5. Execute the schema creation script on the PostgreSQL database:
   ```shell
   psql -h your_postgresql_host -p your_postgresql_port -U your_postgresql_user -d your_postgresql_database -f your_project_directory/your_project_name/schema.sql
   ```

### Data Migration

1. Generate the data migration script:
   ```shell
   ora2pg --project_base your_project_directory --project_name your_project_name --gen_data
   ```

2. Verify the generated data migration script (`your_project_directory/your_project_name/data.sql`). Review it for any necessary modifications.

3. Execute the data migration script on the PostgreSQL database:
   ```shell
   psql -h your_postgresql_host -p your_postgresql_port -U your_postgresql_user -d your_postgresql_database -f your_project_directory/your_project_name/data.sql
   ```

---

## 3. Grails Application Configuration

### Modifying build.gradle

1. Open the `build.gradle` file of your Grails application.

2. Add the PostgreSQL JDBC driver as a dependency:
   ```groovy
   dependencies {
       // Other dependencies
       runtime 'org.postgresql:postgresql:<version>'
   }
   ```

3. Save and close the `build.gradle` file.

### Modifying application.yml

1. Open the `application.yml` file of your Grails application.

2. Modify the database configuration section to use the PostgreSQL database:
   ```yaml
   dataSource:
       pooled: true  # Set it to false in our specific case in Prod
       jmxExport: true
       driverClassName: org.postgresql.Driver
       dialect: org.hibernate.dialect.PostgreSQLDialect
       username: your_postgresql_user
       password: your_postgresql_password
       url: jdbc:postgresql://your_postgresql_host:your_postgresql_port/your_postgresql_database
   ```

3. Save and close the `application.yml` file.

---

## 4. Testing the Web Application

1. Start your Grails application using the appropriate command (e.g., `grails run-app`).

2. Verify that the web application functionalities work fine with the PostgreSQL database. Perform comprehensive testing to ensure all features are functioning correctly.

---

## Pormoting the Standby Server as New Master in case of Disaster

To promote a standby server in PostgreSQL streaming replication as the new master in case of a disaster, you can follow these steps:

1. Verify Standby Server Status: Ensure that the standby server is healthy and up-to-date with the current master. You can check the replication status by running the following command on the standby server:
   ```
   SELECT pg_is_in_recovery();
   ```
   If the result is `false`, it means the server is in standby mode.

2. Stop Replication on Standby Server: On the standby server, stop the replication process by executing the following command as the superuser or a user with the necessary privileges:
   ```
   SELECT pg_wal_replay_pause();
   ```

3. Promote Standby Server: To promote the standby server as the new master, execute the following command as the superuser or a user with the necessary privileges:
   ```
   SELECT pg_promote();
   ```

4. Verify Promotion: Verify that the promotion was successful by checking the status of the server. On the promoted server, run the following command to confirm that it is now the master:
   ```
   SELECT pg_is_in_recovery();
   ```
   The result should be `false`, indicating that the server is no longer in recovery mode.

5. Update Applications: Update the connection configurations of all applications that interact with the database. Modify the connection settings to point to the newly promoted master server.

6. Rebuild Replication Setup: Once the disaster has been resolved, and a new standby server is available, you can rebuild the replication setup by configuring the new standby server to replicate from the new master server.

## Fallback to Original Master in case it comes online

To perform a fallback from the replica (promoted server) to the original master in PostgreSQL streaming replication, you can follow these steps:

1. Verify Original Master Status: Ensure that the original master server is online and reachable. Check its status and confirm that it has recovered from the previous failure or issue.

2. Stop the Promoted Server: On the promoted server (which was previously the standby server and is now the master), stop the PostgreSQL service to prevent any further changes or writes. You can do this by running the appropriate command based on your operating system.

3. Promote the Original Master: On the original master server, execute the following command as the superuser or a user with the necessary privileges to promote it back to being the master:
   ```
   SELECT pg_promote();
   ```

4. Verify Promotion: Check the status of the original master server to confirm that it is now the active master. Run the following command:
   ```
   SELECT pg_is_in_recovery();
   ```
   The result should be `false`, indicating that the server is no longer in recovery mode.

5. Update Applications: Update the connection configurations of all applications that interact with the database. Modify the connection settings to point to the original master server, which is now active again.

6. Reconfigure Replication: Once the fallback is complete, you need to reconfigure the replication setup. Set up the promoted server (which was the master) as a standby server, replicating from the original master server.

7. Monitor and Test: Monitor the system and ensure that the original master is functioning correctly after being promoted. Test the applications to verify that they can connect and perform operations as expected.

It's important to note that the fallback process may result in data loss if changes were made on the promoted server while it was the master. It's recommended to have regular backups and implement a robust disaster recovery strategy to minimize potential data loss.


## Setting up Periodic Backup for the PostgreSQL database
Below is an elaborated step-by-step documentation for setting up periodic backups of your PostgreSQL databases using `pg_basebackup`.

```markdown
# Periodic Backup Setup using pg_basebackup

This guide will walk you through the process of setting up periodic backups for your PostgreSQL databases using `pg_basebackup`. `pg_basebackup` is a utility provided by PostgreSQL for creating physical backups of the entire database cluster.

## Prerequisites

- PostgreSQL installed and running.
- Sufficient disk space available to store the backup files.
- Access to the PostgreSQL server as a user with sufficient privileges.

## Steps

### 1. Choose Backup Location

Decide on a directory to store your backup files. Ensure that the chosen directory has enough disk space to accommodate the backups.

Example:
```bash
BACKUP_DIR="/path/to/backup/directory"
```

### 2. Create Backup Script

Create a backup script that utilizes `pg_basebackup` to perform the backup. This script will be used to automate the backup process.

Create a file named `backup_script.sh` and add the following content:

```bash
#!/bin/bash

# PostgreSQL connection details
PGHOST=<PGSQL_Host_address>
PGPORT="5432"
PGUSER=<username>
PGPASSWORD=<password>

# Backup directory
BACKUP_DIR="/path/to/backup/directory"

# Create a timestamp for the backup filename
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Run pg_basebackup to perform the backup
pg_basebackup -h $PGHOST -p $PGPORT -U $PGUSER -D $BACKUP_DIR/backup_$TIMESTAMP -Ft -z -P

# Check if the backup was successful
if [ $? -eq 0 ]; then
  echo "Backup created successfully."
else
  echo "Backup failed!"
  exit 1
fi
```

Make sure to replace `your_username`, `your_password`, and `/path/to/backup/directory` with your actual PostgreSQL credentials and desired backup directory path.

### 3. Set Permissions and Schedule Backup

Make the backup script executable by running the following command:

```bash
chmod +x backup_script.sh
```

To schedule the backup script to run periodically, you can use cron. Open the cron table editor by running `crontab -e` and add the following line to schedule the backup to run daily at 2:00 AM:

```
0 2 * * * /path/to/backup_script.sh
```

Save the cron table and the backup script will be executed automatically according to the specified schedule.

### 4. Test the Backup

To ensure that the backup script is functioning correctly, manually execute it and verify that the backup files are generated in the specified directory.

```bash
/path/to/backup_script.sh
```

Check the backup directory for the newly created backup file.


