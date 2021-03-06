# shifter

Cassandra migration tool that uses pure CQL to keep the complexity away from migrations and from ORMs.

shifter is based on the idea of simple pure CQL migration files than have a very specific order and should be included in your version control.

## Instalation

```bash
$ pip install cql-shifter
```

## Settings file

The first thing you need to setup before using shifter is creating or pointing your `CASSANDRA_SETTINGS` file.

This is done by defining the `CASSANDRA_SETTINGS` env var to the settings module of your existing application.
That file should expose the following variables that shifter is going to look for:

Variable name             | Required  | Description
---                       | ---       | ---
`CASSANDRA_SEEDS`         | Yes       | A list of IPs of cassandra instances.
`CASSANDRA_KEYSPACE`      | Yes       | The name of your primary keyspace.
`CASSANDRA_PORT`          | No        | Cassandra's port.
`CASSANDRA_CQLVERSION`    | No        | CQL Version. Sometimes it is needed to adjust.
`CASSANDRA_USER`          | No        | Username in case of authentication needed.
`CASSANDRA_PASSWORD`      | No        | Password in case of authentication needed.

You can take a look at `demo/settings.py` to check the defaults. In the case we would like to use that settings file we would have to `$ export CASSANDRA_SETTINGS=demo.settings` and then run any shifter command as usual.

The settings file can be overriden by using the `--settings` flag in some commands (Check out the `--help` for each command).

## Clean start without keyspace schema

If you need to start designing your database from scratch -this means you don't have either db schema nor migration files-, first you need to do all your schema design right in Cassandra (using cqlsh or so).

When you have a *stable* version of your database running and you feel it's a good base, it's time to create out **migration genesis**, which is the base migration schema from which we are going to create any further migrations.

First `cd` into your workspace root (your application root) and create and run  

```bash
$ mkdir migrations
``` 

Then initiate the migration by running

```bash
$ shifter init
```

this will effectively create the migration genesis which is the `migrations/00000.cql` file. This file contains a CQL dump of your current keyspace structure.

Run

```bash
$ shifter migrate
```

To check everything is running smoothly just run `$ shifter status` and you should be prompted an *Already up to date* message. This means we are all set to start making further migrations.

## Clean start with a keyspace schema

If you already have a nice keyspace and you want to start using shifter to control your database changes then the process is much the same as the [previous process](#clean-start-without-keyspace-schema)

## Creating a keyspace from existing migration files

If you already have migrations folder and migration files inside `migrations/` then we need to create a keyspace to match the exact structure of the migration files.

First `cd` into your workspace root (the directory that contains the `migrations/` folder).

Then make sure to update the settings of the `CASSANDRA_SETTINGS` file to match your desired cassandra installation.

Run your first migration:

```bash
$ shifter migrate
```

All done. Start working!

## Creating your first database migration

Once you are all set and you need to perform some change to the database there are 2 ways of doing it.

Migration files are plain old CQL files. In fact you could run those files directly inside cassandra and they will work.

The files are composed by a *header* section for comments, a **UP** section that performs actions to alter the schema UP and **DOWN** section that performs the inverse queries from UP.

### 1 Create a new migration file

This is the preferred method as it is more reliable.

#### 1.1 Create a migration

```bash
$ shifter create "brief description of changes"
```

#### 1.2 Check migration file 

The previous command command will output the creation of a new file -say `00005_create_users_table.cql` - inside the `migrations` folder. Go ahead and open that file in your editor.

#### 1.3 Modify or comment

Feel free to change add/remove comments in the header section of the file `/* ... */`. The file is pretty much self-explainatory. **IMPORTANT** Each command MUST end with `;`.

That's it. Now check the status of your migration:

```bash
$ shifter status
```

you will see it tells you that your migrations folder is 1 movement ahead of your Cassandra database. This means you need to run a migration.

To complete the migration run

```bash
$ shifter migrate
```

**In case you have a syntax error or any of the queries conflict with the schema, the migration will not modify your keyspace in any way.**

This is because shifter creates a keyspace replica at runtime and runs all the migrations in that replica before running it on the real keyspace.

If there was any error in the migration, then the migration will be aborted and the replica will always be deleted.

### 2. Auto generate a migration

If you went ahead and made some changes directly in your database, it means you have effectively outdated the migrations folder!

In this case we need to update your migration folder history, you can do this by running 

```bash
$ shifter auto-update --name "changes description"`
```

This will output a file name that was automatically generated based on the changed you made directly on the database.

That's it! Now check the status 

```bash
$ shifter status
```

and you should see you are up to date!

_TIP:_ If you just want to output the changes that you made against the current keyspace, run

```
$ shifter auto-update --print --name sync
```

*Important:* Auto-update doesn't track column renaming or any changes in the status of a partition key or a clustering key as it would effectively destroy data.

Changes of that nature will need to be tracked manually, that's why it is very importat you check manually every auto-update generated migrations.
