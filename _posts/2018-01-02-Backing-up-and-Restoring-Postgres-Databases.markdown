---
layout: post
date: 2018-01-02 07:00
title: Backing up and Restoring Postgres Databases
---

Postgres is the most advanced open source Relational Database out there. Thousands of companies use it to store their valuable data. So, it is really important to backup this data in order to prevent any kind of data loss. This post will describe how to backup and restore postgres databases.

# **Backing up the database**


Postgres provides a utility called `pg_dump` which is used to create database dumps. It dumps the database into a plain text file, which can then be used to restore the database. `pg_dump` is a simple postgresql client which works by connecting to the postgresql server.

Using `pg_dump` is fairly simple. It is run from the shell like a simple Linux command. The command must be run as a user who is having all the permissions on the database.

    postgres@hostname:~$ pg_dump database_name > backup_file_name

The `pg_dump` client can also backup databases present on remote servers. If no options are provided in the command(like above), it connects to the **localhost**.

To connect to remote hosts, you must pass correct options,

    user@hostname:~$ pg_dump -U username -h remote_host_address -p remote_host_port -d database_name > backup_file_name

By default, it dumps both the schema and data into the dump file. If you only want the schema of the database, then pass `-s` option,

    postgres@hostname:~$ pg_dump -s database_name > backup_file_name

If you only want the data only, then pass the `--data-only` options,

    postgres@hostname:~$ pg_dump --data-only database_name > backup_file_name

Adding more to this specification, you can also get the dump of only a single table,

    postgres@hostname:~$ pg_dump -t table_name database_name > backup_file_name

Remember that it creates a snapshot of the database at the time you run this command. Any further changes will not be included in this snapshot. Therefore, the databases must be backed up regularly.


# **Restoring the database dump**

The data-dump created by `pg_dump` is restored with the help of `psql`. 

    postgres@hostname:~$ psql database_name < backup_file_name

This command does not create the database if it is not present in the server. So, the database being used here, `database_name` must be present in the server before running this command. Also, this database should be empty. If the database is not empty, it will throw errors during restoring stating the respective cause.

One more point which must be remembered during restoring the dump is to have all the users present who own or who were granted access to this database. This typically happens when you are restoring the dump on some different postgres server. For example, suppose you backed up database **database1** from **server1**, and user **user1** owned it. Now you want to restore it to **server2**, and **server2** is not having a user with name **user1** in it, the restore will not be successful. You have to create a user namely **user1** on **server2**.

An evil feature of this restore process is that it will not stop if some error occurred during restoring the dump, it will run until the end and run every line of the dump file. If you want to modify this behaviour, which I think everybody would want to do, you can use the `ON_ERROR_STOP` option,

    postgres@hostname:~$ psql --set ON_ERROR_STOP=on database_name < backup_file

Another common scenario is the partially made database restores. Suppose your dump is being restored, but on 99% of completion, some error occurs in the dump file. This causes the problem of partial restores. And to restore the dump fully again, you would have to create a new empty database or cleanup the partially restored database. Obviously, you do not want this to happen. To stop this behaviour, use the `--single-transaction` option, 

    postgres@hostname:~$ psql --single-transaction database_name < backup_file_name

When using `--single-transaction`, if a error happens, it will rollback the whole restore. 

# **Conclusion**


The `pg_dump` and `psql` is a fairly simple, easy and efficient method to create and restore backups for the databases. This method is more effective than the file system Recovery method. It is simpler than the Point-in-Time Recovery method and is usually suitable for most of the scenarios until and unless your system is really destroyed.

The databases must be backed up regularly and the dumps must be tested regularly. 

