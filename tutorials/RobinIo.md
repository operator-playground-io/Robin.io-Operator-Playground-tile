
###Deploy a PostgreSQL database on Cluster

Create a PostgreSQL database using Helm and Robin Storage. 

When we installed the Robin operator and created a “Robincluster” custom resource definition as part of Install operation, we created and registered a StorageClass named “Robin”. We can now use this StorageClass to create PersistentVolumes and PersistentVolumeClaims for the pods in OpenShift. Using this StorageClass allows us to access the data management capabilities (such as snapshot, clone, backup) provided by Robin Storage.

Add a namespace for Robin to monitor.

```execute
robin namespace add demo
```

Install PostgreSQL chart from bitnami.

```execute
helm install movies bitnami/postgresql -n demo --set persistence.storageClass=robin
```

Run the following command to verify our database called “movies” is deployed and all relevant Kubernetes resources are ready.

```execute
helm list -n demo
```
You should be able to see an output showing the status of your Postgres database.

```
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS   CHART                    APP VERSION
movies  demo            1               2021-04-15 06:08:53.108257288 -0500 CDT deployed postgresql-10.3.15       11.11.0
```

You would also want to make sure Postgres database services are running before proceeding further. Run the following command to verify the services are running.

```execute
kubectl get service -n demo | grep movies
```
You should see an output similar to the following.

```
movies-postgresql            ClusterIP   10.100.187.49   <none>        5432/TCP   3m29s
movies-postgresql-headless   ClusterIP   None            <none>        5432/TCP   3m29s
```


###Add sample data to the PostgreSQL database

```execute
export IP_ADDRESS=$(kubectl get svc movies-postgresql -n demo -o jsonpath={.spec.clusterIP})
```

```execute
export POSTGRES_PASSWORD=$(kubectl get secret --namespace demo movies-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)
```

Let’s create a database “testdb” and connect to “testdb”.


```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -c "CREATE DATABASE testdb;"
```

LIST DATABASES: 

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -c "\l"
```

List of databases
```
  Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
          |          |          |             |             | postgres=CTc/postgres
template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
          |          |          |             |             | postgres=CTc/postgres
testdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(4 rows)
```

For the purpose of this tutorial, let’s create a table named “movies”.

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -c "\c testdb"
```
You are now connected to database "testdb" as user "postgres".

CREATE TABLE:

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "CREATE TABLE movies (movieid TEXT, year INT, title TEXT, genre TEXT);"
```

DESCRIBE TABLE:
```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "\d"

```
List of relations
```
Schema |  Name  | Type  |  Owner
--------+--------+-------+----------
public | movies | table | postgres
(1 row)
```

We need some sample data to perform operations on. Let’s add 9 movies to the “movies” table.

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0360556', 2018, 'Fahrenheit 451', 'Drama');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0365545', 2018, 'Nappily Ever After', 'Comedy');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0427543', 2018, 'A Million Little Pieces', 'Drama');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0432010', 2018, 'The Queen of Sheba Meets the Atom Man', 'Comedy');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0825334', 2018, 'Caravaggio and My Mother the Pope', 'Comedy');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0859635', 2018, 'Super Troopers 2', 'Comedy');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0862930', 2018, 'Dukun', 'Horror');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0891581', 2018, 'RxCannabis: A Freedom Tale', 'Documentary');INSERT INTO movies (movieid, year, title, genre) VALUES ('tt0933876', 2018, 'June 9', 'Horror');"
```
Let’s verify data was added to the “movies” table by running the following command. You should see an output with the “movies” table and the nine rows in it as follows:

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "select * from movies;"
```
```
  movieid  | year |                 title                 |    genre
-----------+------+---------------------------------------+-------------
tt0360556 | 2018 | Fahrenheit 451                        | Drama
tt0365545 | 2018 | Nappily Ever After                    | Comedy
tt0427543 | 2018 | A Million Little Pieces               | Drama
tt0432010 | 2018 | The Queen of Sheba Meets the Atom Man | Comedy
tt0825334 | 2018 | Caravaggio and My Mother the Pope     | Comedy
tt0859635 | 2018 | Super Troopers 2                      | Comedy
tt0862930 | 2018 | Dukun                                 | Horror
tt0891581 | 2018 | RxCannabis: A Freedom Tale            | Documentary
tt0933876 | 2018 | June 9                                | Horror
(9 rows)
```
We now have a PostgreSQL database with a table and some sample data. Now, let’s take a look at the data management capabilities Robin brings, such as taking snapshots, making clones, and creating backups.


###Verify the PostgreSQL Helm release has been registered as an application

To benefit from the data management capabilities, we’ll register our PostgreSQL database with Robin. Doing so will let Robin map and track all resources associated with the Helm release for this PostgreSQL database.

As we have already added demo namespace in robin for the current user (administrator), robin will auto discover the helm apps registered in the demo namespace. verify robin app list and the helm release “movies” is present.

```execute
robin app info movies --status
```
You should see an output similar to this:

```
+-----------------------+----------------------------+--------+---------+
| Kind                  | Name                       | Status | Message |
+-----------------------+----------------------------+--------+---------+
| Secret                | movies-postgresql          | Ready  | -       |
| PersistentVolumeClaim | data-movies-postgresql-0   | Bound  | -       |
| Pod                   | movies-postgresql-0        | Ready  | -       |
| Service               | movies-postgresql          | Ready  | -       |
| Service               | movies-postgresql-headless | Ready  | -       |
| StatefulSet           | movies-postgresql          | Ready  | -       |
+-----------------------+----------------------------+--------+---------+
Key:
  Green: Object is running
  Yellow: Object is potentially down
  Red: Object is down
```

###Snapshot the PostgreSQL Database

If you make a mistake, such as unintentionally deleting important data, you may be able to undo it by restoring a snapshot. Snapshots allow you to restore the state of your application to a point-in-time.

Robin lets you snapshot not just the storage volumes (PVCs) but the entire database application including all its resources such as Pods, StatefulSets, PVCs, Services, ConfigMaps etc. with a single command. To create a snapshot, run the following command.

```execute
robin snapshot create movies --snapname snap9movies --desc "contains 9 movies" --wait
```

Output:

```
Job:   20 Name: K8SApplicationSnapshot State: VALIDATED       Error: 0
Job:   20 Name: K8SApplicationSnapshot State: PROCESSED       Error: 0
Job:   20 Name: K8SApplicationSnapshot State: WAITING         Error: 0
Job:   20 Name: K8SApplicationSnapshot State: COMPLETED       Error: 0
```

Let’s verify we have successfully created the snapshot.

```execute
robin snapshot list --app movies
```
You should see an output similar to this:

```
+----------------------------------+--------+----------+----------+--------------------+
| Snapshot ID                      | State  | App Name | App Kind | Snapshot name      |
+----------------------------------+--------+----------+----------+--------------------+
| e9a5968a9d8111eb9411a7d5a83ebd67 | ONLINE | movies   | helm     | movies_snap9movies |
+----------------------------------+--------+----------+----------+--------------------+
```

We now have a snapshot of our entire database with information of all 9 movies.

###Simulate a user error 

Simulate a user error by deleting some movies from the “movies” table.
 
```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "DELETE from movies where genre = 'Comedy';"
```

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "DELETE from movies where title = 'Dukun';"
```

Check the DB entries now:


```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "select * from movies;"
```

```
  movieid  | year |           title            |    genre    
-----------+------+----------------------------+-------------
 tt0360556 | 2018 | Fahrenheit 451             | Drama
 tt0427543 | 2018 | A Million Little Pieces    | Drama
 tt0891581 | 2018 | RxCannabis: A Freedom Tale | Documentary
 tt0933876 | 2018 | June 9                     | Horror
(4 rows)
```

###Rollback the PostgreSQL database

Let’s run the following command to see the available snapshots:

```execute
robin app info movies
```
You should see an output similar to the following. Note the snapshot id, as we will use it in the next command.

```
Name                              : movies
Kind                              : helm
State                             : ONLINE
Number of repos                   : 0
Number of snapshots               : 1
Number of usable backups          : 0
Number of archived/failed backups : 0
```

Query:
```
{'selectors': [], 'namespace': 'demo', 'apps': ['helm/movies@demo'], 'resources': []}
```

Snapshots:
```
+----------------------------------+--------------------+-------------------+--------+----------------------+
| Id                               | Name               | Description       | State  | Creation Time        |
+----------------------------------+--------------------+-------------------+--------+----------------------+
| e9a5968a9d8111eb9411a7d5a83ebd67 | movies_snap9movies | contains 9 movies | ONLINE | 30 Sep 2020 07:40:27 |
+----------------------------------+--------------------+-------------------+--------+----------------------+
```
Now, let’s rollback to the point where we had 9 movies :

```execute
robin app restore movies --snapshotid SNAPSHOT_ID_HERE --wait 
```
You should see an output similar to the following:

```
Job:   26 Name: K8SApplicationRollback State: PROCESSED       Error: 0
Job:   26 Name: K8SApplicationRollback State: PREPARED        Error: 0
Job:   26 Name: K8SApplicationRollback State: AGENT_WAIT      Error: 0
Job:   26 Name: K8SApplicationRollback State: COMPLETED       Error: 0
```
To verify we have rolled back to 9 movies in the “movies” table, run the following command.

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "select * from movies;"
```
```
  movieid  | year |                 title                 |    genre
-----------+------+---------------------------------------+-------------
tt0360556 | 2018 | Fahrenheit 451                        | Drama
tt0365545 | 2018 | Nappily Ever After                    | Comedy
tt0427543 | 2018 | A Million Little Pieces               | Drama
tt0432010 | 2018 | The Queen of Sheba Meets the Atom Man | Comedy
tt0825334 | 2018 | Caravaggio and My Mother the Pope     | Comedy
tt0859635 | 2018 | Super Troopers 2                      | Comedy
tt0862930 | 2018 | Dukun                                 | Horror
tt0891581 | 2018 | RxCannabis: A Freedom Tale            | Documentary
tt0933876 | 2018 | June 9                                | Horror
(9 rows)
```

We have successfully rolled back to our original state with 9 movies!

###Clone the PostgreSQL Database

Robin lets you clone not just the storage volumes (PVCs) but the entire database application including all its resources such as Pods, StatefulSets, PVCs, Services, ConfigMaps, etc. with a single command.

Application cloning improves the collaboration across Dev/Test/Ops teams. Teams can share applications and data quickly, reducing the procedural delays involved in re-creating environments. Each team can work on their clone without affecting other teams. Clones are useful when you want to run a report on a database without affecting the source database application, or for performing UAT tests or for validating patches before applying them to the production database, etc.

Robin clones are ready-to-use “thin copies” of the entire app/database, not just storage volumes. Thin-copy means that data from the snapshot is NOT physically copied, therefore clones can be made very quickly. Robin clones are fully-writable and any modifications made to the clone are not visible to the source app/database.

To create a clone from the existing snapshot created above, run the following command. Use the snapshot id we retrieved above.

```copy
robin app create from-snapshot movies-clone SNAPSHOT_ID_HERE --wait
```
You should see output similar to the following:

```

Job:  137 Name: K8SApplicationClone  State: VALIDATED       Error: 0
Job:  137 Name: K8SApplicationClone  State: PREPARED        Error: 0
Job:  137 Name: K8SApplicationClone  State: AGENT_WAIT      Error: 0
Job:  137 Name: K8SApplicationClone  State: FINALIZED       Error: 0
Job:  137 Name: K8SApplicationClone  State: COMPLETED       Error: 0
```

Let’s verify Robin has cloned all relevant Kubernetes resources.

```execute
kubectl get all -n demo | grep "movies-clone"
```
You should see an output similar to below.


```
pod/movies-clone-movies-postgresql-0   1/1     Running   0          94s
service/movies-clone-movies-postgresql            ClusterIP   172.30.149.75    <none>        5432/TCP    109s
service/movies-clone-movies-postgresql-headless   ClusterIP   None             <none>        5432/TCP    109s
statefulset.apps/movies-clone-movies-postgresql   1/1     109s
```

Notice that Robin automatically clones the required Kubernetes resources, not just storage volumes (PVCs), that are required to stand up a fully-functional clone of our database. After the clone is complete, the cloned database is ready for use.

Get Service IP address of our postgresql database clone, and note the IP address.

```execute
export IP_ADDRESS=$(kubectl get svc movies-clone-movies-postgresql -n demo -o jsonpath={.spec.clusterIP})
```

Get Password of our postgresql database clone from Kubernetes Secret

```execute
export POSTGRES_PASSWORD=$(kubectl get secret movies-clone-movies-postgresql -n demo -o jsonpath="{.data.postgresql-password}" | base64 --decode;)
```

To verify we have successfully created a clone of our PostgreSQL database, run the following command. You should see an output similar to the following:


```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "select * from movies;"
```
```
  movieid  | year |                 title                 |    genre
-----------+------+---------------------------------------+-------------
tt0360556 | 2018 | Fahrenheit 451                        | Drama
tt0365545 | 2018 | Nappily Ever After                    | Comedy
tt0427543 | 2018 | A Million Little Pieces               | Drama
tt0432010 | 2018 | The Queen of Sheba Meets the Atom Man | Comedy
tt0825334 | 2018 | Caravaggio and My Mother the Pope     | Comedy
tt0859635 | 2018 | Super Troopers 2                      | Comedy
tt0862930 | 2018 | Dukun                                 | Horror
tt0891581 | 2018 | RxCannabis: A Freedom Tale            | Documentary
tt0933876 | 2018 | June 9                                | Horror
(9 rows)
```
We have successfully created a clone of our original PostgreSQL database, and the cloned database also has a table called “movies” with 9 rows, just like the original.

Now, let’s make changes to the clone and verify the original database remains unaffected by changes to the clone. Let’s delete the movie called “Super Troopers 2”.

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "DELETE from movies where title = 'Super Troopers 2';"
```
Let’s verify the movie has been deleted. You should see an output similar to the following with 8 movies.


```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "select * from movies;"
```
```
  movieid  | year |                 title                 |    genre
-----------+------+---------------------------------------+-------------
tt0360556 | 2018 | Fahrenheit 451                        | Drama
tt0365545 | 2018 | Nappily Ever After                    | Comedy
tt0427543 | 2018 | A Million Little Pieces               | Drama
tt0432010 | 2018 | The Queen of Sheba Meets the Atom Man | Comedy
tt0825334 | 2018 | Caravaggio and My Mother the Pope     | Comedy
tt0862930 | 2018 | Dukun                                 | Horror
tt0891581 | 2018 | RxCannabis: A Freedom Tale            | Documentary
tt0933876 | 2018 | June 9                                | Horror
(8 rows)
```
Now, let’s connect to our original PostgreSQL database and verify it is unaffected.

Get Service IP address of our postgresql database.

```execute
export IP_ADDRESS=$(kubectl get svc movies-postgresql -n demo -o jsonpath={.spec.clusterIP})
```

Get Password of our original postgre database from Kubernetes Secret.

```execute
export POSTGRES_PASSWORD=$(kubectl get secret --namespace demo movies-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode;)
```
To verify that our PostgreSQL database is unaffected by changes to the clone, run the following command.

Let’s connect to “testdb” and check record and you should see an output similar to the following, with all 9 movies present:

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "select * from movies;"
```
```
  movieid  | year |                 title                 |    genre
-----------+------+---------------------------------------+-------------
tt0360556 | 2018 | Fahrenheit 451                        | Drama
tt0365545 | 2018 | Nappily Ever After                    | Comedy
tt0427543 | 2018 | A Million Little Pieces               | Drama
tt0432010 | 2018 | The Queen of Sheba Meets the Atom Man | Comedy
tt0825334 | 2018 | Caravaggio and My Mother the Pope     | Comedy
tt0859635 | 2018 | Super Troopers 2                      | Comedy
tt0862930 | 2018 | Dukun                                 | Horror
tt0891581 | 2018 | RxCannabis: A Freedom Tale            | Documentary
tt0933876 | 2018 | June 9                                | Horror
(9 rows)
```
This means we can work on the original PostgreSQL database and the cloned database simultaneously without affecting each other. This is valuable for collaboration across teams where each team needs to perform unique set of operations.

To see a list of all clones created by Robin run the following command:

```execute
robin app list --app-types CLONE
```

Now let’s delete the clone. Clone is just any other Robin app so it can be deleted using the native ‘app delete’ command show below.

```execute
robin app delete movies-clone -y --force --wait
```
The output should be similar to the following:

```
Job:  138 Name: K8SAppDelete         State: PROCESSED       Error: 0
Job:  138 Name: K8SAppDelete         State: PREPARED        Error: 0
Job:  138 Name: K8SAppDelete         State: AGENT_WAIT      Error: 0
Job:  138 Name: K8SAppDelete         State: FINALIZED       Error: 0
Job:  138 Name: K8SAppDelete         State: COMPLETED       Error: 0
```

###Backup the PostgreSQL Database to AWS S3

Robin elevates the experience from backing up just storage volumes (PVCs) to backing up entire applications/databases, including their metadata, configuration, and data.

A backup is a full copy of the application snapshot that resides on completely different storage media than the application’s data. Therefore, backups are useful to restore an entire application from an external storage media in the event of catastrophic failures, such as disk errors, server failures, or entire data centers going offline, etc. (This is assuming your backup doesn’t reside in the data center that is offline, of course.)

Let’s now backup our database to an external secondary storage repository (repo). Snapshots (metadata + configuration + data) are backed up into the repo.

Robin enables you to back up your Kubernetes applications to AWS S3 or Google GCS ( Google Cloud Storage). In this demo we will use AWS S3 to create the backup.

Before we proceed, we need to create an S3 bucket and get access parameters for it. Follow the documentation here.

Let’s first register an AWS repo with Robin via the following command:

```execute
 robin repo register pgsqlbackups s3://robin-pgsql/pgsqlbackups awstier.json readwrite --wait
```

The following should be displayed when the above command is run:

```
Job:  139 Name: StorageRepoAdd       State: PROCESSED       Error: 0
Job:  139 Name: StorageRepoAdd       State: COMPLETED       Error: 0
```
Let’s confirm that our secondary storage repository is successfully registered:

```execute
robin repo list
```

```
+--------------+--------+----------------------+--------------+-------------+---------------+-------------+
| Name         | Type   | Owner/Tenant         | BackupTarget | Bucket      | Path          | Permissions |
+--------------+--------+----------------------+--------------+-------------+---------------+-------------+
| pgsqlbackups | AWS_S3 | admin/Administrators | 1            | robin-pgsql | pgsqlbackups/ | readwrite   |
+--------------+--------+----------------------+--------------+-------------+---------------+-------------+
```
Let’s attach this repo to our app so that we can backup its snapshots there:

```execute
robin app attach-repo movies pgsqlbackups --wait
```
Let’s confirm that our secondary storage repository is successfully attached to app:

```execute
robin app info movies
```
You should see an output similar to the following:

```
Name                              : movies
Kind                              : helm
State                             : ONLINE
Number of repos                   : 1
Number of snapshots               : 1
Number of usable backups          : 0
Number of archived/failed backups : 0

```

Query:
```
{'namespace': 'demo', 'apps': ['helm/movies@demo'], 'resources': [], 'selectors': []}
```
```
Repos:
+--------------+-------------+---------------+------------+
| Name         | Bucket      | Path          | Permission |
+--------------+-------------+---------------+------------+
| pgsqlbackups | robin-pgsql | pgsqlbackups/ | readwrite  |
+--------------+-------------+---------------+------------+
```
```
Snapshots:
+----------------------------------+--------------------+-------------------+--------+----------------------+
| Id                               | Name               | Description       | State  | Creation Time        |
+----------------------------------+--------------------+-------------------+--------+----------------------+
| e9a5968a9d8111eb9411a7d5a83ebd67 | movies_snap9movies | contains 9 movies | ONLINE | 30 Sep 2020 07:40:27 |
+----------------------------------+--------------------+-------------------+--------+----------------------+
```

Lets take the backup of the snapshot to the remote S3 repo.

```execute
robin backup create movies pgsqlbackups --snapshotid Your_snaphsot_ID --backupname Name_of_Backup --wait
```
You should see an output similar to the following:

```
Creating app backup 'movies_backup' from snapshot '13cc8da2031a11eb99a99b354219f676'
Job:  142 Name: K8SApplicationBackup State: PROCESSED       Error: 0
Job:  142 Name: K8SApplicationBackup State: AGENT_WAIT      Error: 0
Job:  142 Name: K8SApplicationBackup State: COMPLETED       Error: 0
```
Let’s also confirm that backup has been copied to remote S3 repo:

```execute
robin repo contents pgsqlbackups
```
You should see an output similar to the following:

```
+----------------------------------+------------+--------------+----------------------+--------+--------------------+----------+
| BackupID                         | ZoneID     | RepoName     | Owner/Tenant         | App    | Snapshot           | Imported |
+----------------------------------+------------+--------------+----------------------+--------+--------------------+----------+
| 60fe30f4032011eb9ac3e5c47d7a9357 | 1601408284 | pgsqlbackups | admin/Administrators | movies | movies_snap9movies | False    |
+----------------------------------+------------+--------------+----------------------+--------+--------------------+----------+
```
The snapshot has now been backed up into our AWS S3 bucket. Let’s note the “BackupID”, because we will need it to restore the database in the next step.

###Restore the PostgreSQL Database

Let’s simulate a system failure where you lose local data. First, let’s delete the snapshot locally.


```execute
robin snapshot delete SNAPSHOT_ID_HERE --wait
```

```
Job:   143 Name: K8SSnapshotDelete    State: PREPARED        Error: 0
Job:   143 Name: K8SSnapshotDelete    State: COMPLETED       Error: 0
```
Now let’s simulate a data loss situation by deleting all data from the “movies” table and verify all data is lost.



```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "DELETE from movies;"
```

```
DELETE 9
```

```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "SELECT * from movies;"

```
```
movieid | year | title | genre
---------+------+-------+-------
(0 rows)
```

We will now use our backed-up snapshot on S3 to restore data we just lost.

Now let’s restore snapshot from the backup in cloud and rollback our application to that snapshot via the following command:
```execute
robin app restore movies --backupid Your_Backup_ID --wait`
```
You should see output similar to the following:

```
Job:  144 Name: K8SApplicationRollback State: VALIDATED       Error: 0
Job:  144 Name: K8SApplicationRollback State: PREPARED        Error: 0
Job:  144 Name: K8SApplicationRollback State: AGENT_WAIT      Error: 0
Job:  144 Name: K8SApplicationRollback State: COMPLETED       Error: 0
```

Remember, we had deleted the local snapshot of our data. Let’s verify the above command has pulled the snapshot stored in the cloud. Run the following command:

```execute
robin snapshot list --app movies
```
You should see output similar to the following:

```
+----------------------------------+--------+----------+----------+--------------------+
| Snapshot ID                      | State  | App Name | App Kind | Snapshot name      |
+----------------------------------+--------+----------+----------+--------------------+
| e9a5968a9d8111eb9411a7d5a83ebd67 | ONLINE | movies   | helm     | movies_snap9movies |
+----------------------------------+--------+----------+----------+--------------------+
```
Let’s verify all 9 rows are restored to the “movies” table by running the following command:


If you don't see a command prompt, try pressing enter.
```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "SELECT * from movies;"
```
```
  movieid  | year |                 title                 |    genre
-----------+------+---------------------------------------+-------------
tt0360556 | 2018 | Fahrenheit 451                        | Drama
tt0365545 | 2018 | Nappily Ever After                    | Comedy
tt0427543 | 2018 | A Million Little Pieces               | Drama
tt0432010 | 2018 | The Queen of Sheba Meets the Atom Man | Comedy
tt0825334 | 2018 | Caravaggio and My Mother the Pope     | Comedy
tt0859635 | 2018 | Super Troopers 2                      | Comedy
tt0862930 | 2018 | Dukun                                 | Horror
tt0891581 | 2018 | RxCannabis: A Freedom Tale            | Documentary
tt0933876 | 2018 | June 9                                | Horror
(9 rows)
```

As you can see, we can restore the database to a desired state in the event of data corruption. We simply pull the backup from the cloud and use it to restore the database.

###Create PostgreSQL Database from the backup

Since we have taken backup of PostgreSQL database, we can create a new app using the backup and verify the data integrity of the postgreSQL database.

```execute
robin app create from-backup <app_name> <Your_backupID> --wait
```

```
Job: 1093 Name: K8SApplicationCreate State: VALIDATED       Error: 0
Job: 1093 Name: K8SApplicationCreate State: PREPARED        Error: 0
Job: 1093 Name: K8SApplicationCreate State: AGENT_WAIT      Error: 0
Job: 1093 Name: K8SApplicationCreate State: COMPLETED       Error: 0
```










```
+-----------------------+---------------------------------------+--------+---------+
| Kind                  | Name                                  | Status | Message |
+-----------------------+---------------------------------------+--------+---------+
| Secret                | movies-bkp-movies-postgresql          | Ready  | -       |
| PersistentVolumeClaim | data-movies-bkp-movies-postgresql-0   | Bound  | -       |
| Pod                   | movies-bkp-movies-postgresql-0        | Ready  | -       |
| Service               | movies-bkp-movies-postgresql          | Ready  | -       |
| Service               | movies-bkp-movies-postgresql-headless | Ready  | -       |
| StatefulSet           | movies-bkp-movies-postgresql          | Ready  | -       |
+-----------------------+---------------------------------------+--------+---------+
```
Lets verify the contents of the postgreSQL database app.

```execute
export POSTGRES_PASSWORD=$(kubectl get secret movies-bkp-movies-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode;)
```
Connect to “testdb” and check record and you should see an output similar to the following, with all 9 movies present:


```execute
PGPASSWORD="$POSTGRES_PASSWORD" psql -h $IP_ADDRESS -U postgres -d testdb -c "SELECT * from movies;"
```
```
  movieid  | year |                 title                 |    genre
-----------+------+---------------------------------------+-------------
tt0360556 | 2018 | Fahrenheit 451                        | Drama
tt0365545 | 2018 | Nappily Ever After                    | Comedy
tt0427543 | 2018 | A Million Little Pieces               | Drama
tt0432010 | 2018 | The Queen of Sheba Meets the Atom Man | Comedy
tt0825334 | 2018 | Caravaggio and My Mother the Pope     | Comedy
tt0859635 | 2018 | Super Troopers 2                      | Comedy
tt0862930 | 2018 | Dukun                                 | Horror
tt0891581 | 2018 | RxCannabis: A Freedom Tale            | Documentary
tt0933876 | 2018 | June 9                                | Horror
(9 rows)
```


