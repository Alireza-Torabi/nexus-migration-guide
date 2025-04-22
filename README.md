
# Educational Booklet: Migrating Nexus Repository from OrientDB to H2 in Docker Compose

## Table of Contents
1. [Introduction](#1-introduction)
2. [Preparation](#2-preparation)
   - [Backup Process](#backup-process)
   - [Confirming the Current Database (OrientDB)](#confirming-the-current-database-orientdb)
3. [Migration Process](#3-migration-process)
   - [Downloading the Migrator Tool](#downloading-the-migrator-tool)
   - [Running the Migration](#running-the-migration)
   - [Moving the H2 Database](#moving-the-h2-database)
4. [Post-Migration Configuration](#4-post-migration-configuration)
   - [Enabling H2 in Configuration](#enabling-h2-in-configuration)
5. [Starting Nexus and Verifying Migration](#5-starting-nexus-and-verifying-migration)
   - [Restarting Nexus with H2](#restarting-nexus-with-h2)
   - [Verifying Data Integrity](#verifying-data-integrity)
6. [Preparing for Nexus 3.71.0+ Upgrade](#6-preparing-for-nexus-3710-upgrade)
7. [Backup & Clean-Up](#7-backup--clean-up)
8. [Troubleshooting and Additional Tips](#8-troubleshooting-and-additional-tips)
9. [Conclusion](#9-conclusion)

---

## 1. Introduction

Nexus Repository Manager, a tool widely used for managing repositories (like Maven, npm, Docker, etc.), has supported OrientDB as its embedded database in previous versions (before 3.71.0). However, in Nexus 3.71.0 and beyond, OrientDB will no longer be supported. The repository data will now rely on **H2** (or PostgreSQL if chosen) as the new database.

This booklet provides a **step-by-step guide** for migrating your Nexus Repository from OrientDB to H2, especially in a Docker Compose setup. By following this guide, you'll be prepared to upgrade to Nexus 3.71.0 and ensure your data is preserved.

---

## 2. Preparation

### Backup Process
Before migrating, **backing up your data** is crucial to prevent data loss. Follow these steps:

1. **Exporting OrientDB Data**:
   - **Navigate to Nexus UI** and go to **Administration → System → Tasks**.
   - Find and run the **“Admin - Export Database for backup”** task. Set the backup location to a safe directory under `/nexus-data` (e.g., `/nexus-data/backup`).
   - This creates backup files with the `.bak` extension, including important data (like `component-<timestamp>.bak`, `config-<timestamp>.bak`, and `security-<timestamp>.bak`).
   
2. **Backup Blob Storage** (optional but recommended):
   - Consider copying or snapshotting the entire `nexus-data` volume as an additional precaution. This will ensure that your repository artifacts are safe, apart from just the database itself.

3. **Verify the Backup**:
   - After the backup completes, verify that the `.bak` files exist in the backup directory. Also, check the **Task History** for success confirmation.

### Confirming the Current Database (OrientDB)

To ensure you’re using OrientDB, check the following:
- **Backup Files**: If you see `.bak` files from the previous step, your data is on OrientDB.
- **Nexus Properties**: Check the `nexus.properties` file in `/nexus-data/etc/`. If it doesn't contain `nexus.datastore.enabled=true`, you're using OrientDB.
- **Database Files**: Ensure there is no `nexus.mv.db` in `/nexus-data/db` at this point (this file will be created after migration to H2).

---

## 3. Migration Process

### Downloading the Migrator Tool

Sonatype provides the **Database Migrator** tool for migrating from OrientDB to H2. Here’s how to get it:

1. **Download the migrator tool for Nexus 3.70.4**:
   - **Inside the Nexus container**, use the following command to download the migration tool:
     ```bash
     docker exec -it <nexus_container_name> /bin/bash
     cd /nexus-data/backup
     curl -O https://download.sonatype.com/nexus/nxrm3-migrator/nexus-db-migrator-3.70.4-02.jar
     ```
   - If your host machine supports running Java, you can download the tool from [Sonatype's website](https://download.sonatype.com/nexus/nxrm3-migrator/).

### Running the Migration

Once you’ve downloaded the migrator tool:

1. **Stop Nexus**:
   - Ensure Nexus is not running. You can stop it via Docker Compose:
     ```bash
     docker-compose stop nexus
     ```

2. **Run the Migrator Tool**:
   - From the `/nexus-data/backup` directory (where your `.bak` files are stored), run the migration command:
     ```bash
     java -Xmx4G -Xms4G -XX:+UseG1GC -XX:MaxDirectMemorySize=4G           -jar nexus-db-migrator-3.70.4-02.jar --migration_type=h2
     ```
   - The tool will convert the OrientDB data to the H2 format. Make sure to confirm when prompted by the tool. It will create a `nexus.mv.db` file in your current directory (where your `.bak` files are).

### Moving the H2 Database

1. **Move the H2 Database**:
   - Once the migration is complete, move the generated `nexus.mv.db` file into the `/nexus-data/db/` directory:
     ```bash
     mv /nexus-data/backup/nexus.mv.db /nexus-data/db/nexus.mv.db
     ```

---

## 4. Post-Migration Configuration

### Enabling H2 in Configuration

To ensure Nexus uses H2:
1. **Edit the `nexus.properties` file** in `/nexus-data/etc/` and add the following:
   ```properties
   nexus.datastore.enabled=true
   ```
   This property tells Nexus to use H2 instead of OrientDB for the database.

2. **Ensure No OrientDB Flags**:
   - The OrientDB database will be ignored if `nexus.datastore.enabled=true` is set.

---

## 5. Starting Nexus and Verifying Migration

### Restarting Nexus with H2

1. **Start Nexus**:
   - Restart the Nexus container using Docker Compose:
     ```bash
     docker-compose up -d nexus
     ```
   - Alternatively, if you stopped Nexus inside the container earlier, you can start it again from within:
     ```bash
     /opt/sonatype/nexus/bin/nexus start
     ```

2. **Monitor Logs**:
   - Watch the logs using:
     ```bash
     docker-compose logs -f nexus
     ```
   - Ensure there are no errors related to the database. Nexus should now be operating on H2 and not OrientDB.

---

## 6. Verifying Data Integrity

After Nexus restarts, it's critical to verify that the migration was successful and your data is intact:
   
- **Verify Repositories**: Check all repositories in the Nexus UI. Ensure all components, artifacts, and metadata appear as expected.
- **Check Security Data**: Verify that user accounts and security roles are intact.
- **Validate Search and Indexing**: Run some searches to confirm that search indexes have been rebuilt properly.
- **Perform Blob Store Reconciliation**: Run a task to reconcile the database with the blob store, ensuring all files are correctly referenced.

---

## 7. Preparing for Nexus 3.71.0+ Upgrade

With the migration to H2 complete, you are ready to upgrade Nexus to version 3.71.0. 

1. **Update Docker Compose**:
   - In your `docker-compose.yml`, update the image to the latest 3.71.0+ version:
     ```yaml
     services:
       nexus:
         image: sonatype/nexus3:3.71.0
     ```

2. **Run the Upgrade**:
   - Pull the new image and restart:
     ```bash
     docker-compose pull nexus
     docker-compose up -d nexus
     ```
   - Nexus should now be using H2, and you’ll be able to use all the features of version 3.71.0 and beyond.

---

## 8. Backup & Clean-Up

Once you confirm that Nexus 3.71 is running well, it’s a good idea to:
- **Backup**: Take a fresh backup of the new H2 database.
- **Clean-Up**: If necessary, clean up old OrientDB files and backups. Ensure to keep them in case you need to revert.

---

## 9. Troubleshooting and Additional Tips

- **Migration Errors**: If there are any errors during the migration, check the logs and ensure the migration command was run properly. In case of failure, you can restore from the backup and attempt the migration again.
- **Data Integrity**: If any repositories or components are missing, check the logs for any errors during the migration process. You may also need to run additional reconciliation tasks in the Nexus UI to fix indexing issues.

---

## 10. Conclusion

Congratulations! You’ve successfully migrated your Nexus Repository from OrientDB to H2 using Docker Compose. With this process, you can now safely upgrade to Nexus 3.71.0 or later. Regular backups, configuration checks, and post-migration validation will ensure your Nexus system remains healthy as you continue using it for your repository management needs.

---

### Keep Learning
Feel free to explore more about Nexus, its features, and how to manage your repository effectively through Sonatype’s [official documentation](https://help.sonatype.com/).
