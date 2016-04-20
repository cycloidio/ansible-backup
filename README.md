ansible-backup
==============

This role install and configure backups for several applications like elasticsearch and mongodb.

No global requirements. It will depend of what type of backup you want. See specific Requirements section for each backup type.

The only things you need is to specify `backup_type` variable to define what kind of backup you want to setup.

elasticsearch
=============

For now on elasticsearch don't provide a "retention policy" feature for snapshots. So we are using the python module `elasticsearch-curator` and/or s3 bucket lifecyle policy to do the job (https://www.elastic.co/guide/en/elasticsearch/client/curator/index.html).

Elasticsearch index snapshot process is incremental. In the process of making the index snapshot Elasticsearch analyses the list of the index files that are already stored in the repository and copies only files that were created or changed since the last snapshot. When a snapshot is deleted from a repository, Elasticsearch deletes all files that are associated with the deleted snapshot and not used by any other snapshots.  (https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html)

We choose to have one distinct full backup by week. So by default the backup behavior is : 

  * Create a directory inside the bucket with name : `%Y-week-%V` (year number and ISO week number).
  * Do the backup.
  * Next backup will create new directory inside the bucket s3 if we change year or month.

So if you configure the cron to run snapshot command every days, the expected behavior is : 

  * Each day elasticsearch do a new incremental snapshot.
  * Each week, the script will create a new directory. So we do a new independant full backup each week.

Additionally to this, you can use the `backup_es_retention` to limite the number of snapshot by directory (`backup_es_s3_path`). For example if you do one backup per hour
and want to keep only hour snapshot for the last 3 days.


Requirements
------------

We are using `elasticsearch-cloud-aws` plugin to backup on s3 bucket. So ensure configured AWS credentials for this plugin (https://github.com/elastic/elasticsearch-cloud-aws#generic-configuration)

Exemple with `aloysius.elasticsearch` ansible role :

```
    - role: aloysius.elasticsearch
      es_etc:
        cloud.aws.region: "..."
        cloud.aws.access_key: "..."
        cloud.aws.secret_key: "..."
```

The default cron command for backup use `cronlock`. For Cycloid `cronlock` is provided by the `ansible-base` role. If you don't want `cronlock`, just override the `backup_es_cron` variable.

Last thing, to configure the s3 bucket lifecycle policy, this playbook use `aws-cli`. You should ensure the ssh user used to run the playbook is able to do aws commands on the bucket.

Role Variables
--------------

**Required**

Name and region of the bucket to use for backups on s3

    backup_es_bucket_name: foo
    backup_es_bucket_region: us-east-1

**Optional**

Enable the validate task callback to check if the role setup succeded

    validate_task: true


Limite the age of snapshot inside the `backup_es_s3_path` directory. The retention command is called after each backups. Default `0` disabled (https://www.elastic.co/guide/en/elasticsearch/client/curator/current/snapshots-subcommand.html)

    backup_es_retention:
      unit: days
      value: 0

The retention part if you have a dynamic `backup_es_s3_path` can be handle with  s3 lifecycle policy : http://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-lifecycle.html

    # Example one : Every file in the s3 bucket will be removed after 365 days.
    backup_s3_expire_policy:
      permanently_delete: 365
      glacier_transition: 0

    # Example two : Every file myprefix* will be send on glacier after 60 days. And deleted from glacier after 365 days.
    backup_s3_expire_policy:
      prefix: "myprefix"
      permanently_delete: 365
      glacier_transition: 60


S3 subdirectory name. You can use bash command inside with `'`. backup script ensure the directory exist at each backup. If not it will create this directory. The default example will ensure we have a new directory every week.

    backup_es_s3_path: "elasticsearch-'$(date +%Y-week-%V)'"


Configure the cron commands for the backup job.

    backup_es_cron:
        minute: "0"
        hour: "3"


Command for backup and retention inside the elasticsearch-backup.sh script :

    backup_es_backup_cmd: "/usr/local/bin/curator snapshot --ignor_...
    backup_es_retention_cmd: "/usr/local/bin/curator delete snapsho...


**Example :**

```
  roles:
    - role: ansible-backup
      validate_task: true # Run the validator
      backup_type: elasticsearch
      backup_es_bucket_name: "{{ backup_es_bucket_name }}"
      backup_es_bucket_region: "{{ aws_default_region }}"

```

Mongodb
=======

Based on https://docs.mongodb.org/v3.0/tutorial/backup-small-sharded-cluster-with-mongodump/
You should follow https://docs.mongodb.org/v3.0/administration/backup-sharded-clusters/ if you have a big shard cluster.

Requirements
------------

To upload mongodb dump on s3 bucket, this role use `aws-cli`. It's important that you configured `aws-cli` for the user who will launch the cron backup command.
For example you can use a `.boto` file. (http://boto.cloudhackers.com/en/latest/boto_config_tut.html)

The default cron command for backup use `cronlock`. For Cycloid `cronlock` is provided by the `ansible-base` role. If you don't want `cronlock`, just override the `backup_mongo_cron` variable.

Role Variables
--------------

**Required**

backup_mongo_bucket_name


**Optional**

Enable the validate task callback to check if the role setup succeded

    validate_task: true

Backup cron for mongodb dump

    backup_mongo_cron:
      name: snapshot
      job: "/usr/bin/cronlock ...
      minute: "0"
      hour: "3"
      day: "*"
      month: "*"
      weekday: "*"


Do a lock during mongodump (https://docs.mongodb.org/manual/reference/method/db.fsyncLock/). Default `false`

    backup_mongo_with_lock: false


Configure directory on the machine and s3 for the backup. Local backup is deleted after the upload in s3.

    backup_mongo_local_path: /home/admin/backups/mongodb
    backup_mongo_s3_path: mongo

Configure the mongodb backup script commands :

    backup_mongo_lock_cmd: /usr/bin/mongo admin --eval "printjson(db.fsyncLock())"
    backup_mongo_unlock_cmd: /usr/bin/mongo admin --eval "printjson(db.fsyncUnlock())"
    backup_mongo_cmd: /usr/bin/mongodump --oplog --out {{ backup_mongo_local_path }}


Configure backup retention handled by s3 bucket lifecycle policy : http://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-lifecycle.html

    # Example one : Every file in the s3 bucket will be removed after 365 days.
    backup_s3_expire_policy:
      permanently_delete: 365
      glacier_transition: 0

    # Example two : Every file myprefix* will be send on glacier after 60 days. And deleted from glacier after 365 days.
    backup_s3_expire_policy:
      prefix: "myprefix"
      permanently_delete: 365
      glacier_transition: 60



**Example :**

```
    - role: ansible-backup
      validate_task: true # Run the validator
      backup_type: mongodb
      backup_mongo_bucket_name: "{{ backup_mongo_bucket_name }}"
```

Postgresql
==========


Requirements
------------

To upload postgresql dump on s3 bucket, this role use `aws-cli`. It's important that you configured `aws-cli` for the user who will launch the cron backup command.
For example you can use a `.boto` file. (http://boto.cloudhackers.com/en/latest/boto_config_tut.html)

The default cron command for backup use `cronlock`. For Cycloid `cronlock` is provided by the `ansible-base` role. If you don't want `cronlock`, just override the `backup_postgresql_cron` variable.

Role Variables
--------------

**Required**

Name of the bucket to use for the upload backup

  * `backup_postgresql_bucket_name`

Credentials to connect to postgresql

  * `backup_postgresql_password`
  * `backup_postgresql_host`
  * `backup_postgresql_user`


**Optional**

Enable the validate task callback to check if the role setup succeded

    validate_task: true

Backup cron for postgresqldb dump

    backup_postgresql_cron:
      name: snapshot
      job: "/usr/bin/cronlock ...
      minute: "0"
      hour: "3"
      day: "*"
      month: "*"
      weekday: "*"


Configure directory on the machine and s3 for the backup. Local backup is deleted after the upload in s3.

    backup_postgresql_local_path: /home/admin/backups/postgresql
    backup_postgresql_s3_path: postgresql

Configure the postgresql backup script commands :

    backup_postgresql_cmd: PGPASSWORD={{ backup_postgresql_password }} pg_dump -b -h {{ backup_postgresql_host }} -U {{ backup_postgresql_user }} -Fc $DB > {{ backup_postgresql_local_path }}/$DB.bin


Configure backup retention handled by s3 bucket lifecycle policy : http://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-lifecycle.html

    # Example one : Every file in the s3 bucket will be removed after 365 days.
    backup_s3_expire_policy:
      permanently_delete: 365
      glacier_transition: 0

    # Example two : Every file myprefix* will be send on glacier after 60 days. And deleted from glacier after 365 days.
    backup_s3_expire_policy:
      prefix: "myprefix"
      permanently_delete: 365
      glacier_transition: 60



**Example :**

```
    - role: ansible-backup
      validate_task: true # Run the validator
      backup_type: postgresql
      backup_postgresql_bucket_name: "{{ backup_postgresql_bucket_name }}"
      backup_postgresql_password: password
      backup_postgresql_host: 1.2.3.4
      backup_postgresql_user: foo
```

Mysql
==========


Requirements
------------

To upload mysql dump on s3 bucket, this role use `aws-cli`. It's important that you configured `aws-cli` for the user who will launch the cron backup command.
For example you can use a `.boto` file. (http://boto.cloudhackers.com/en/latest/boto_config_tut.html)

The default cron command for backup use `cronlock`. For Cycloid `cronlock` is provided by the `ansible-base` role. If you don't want `cronlock`, just override the `backup_postgresql_cron` variable.

Role Variables
--------------

**Required**

Name of the bucket to use for the upload backup

  * `backup_mysql_bucket_name`

Credentials to connect to postgresql

  * `backup_mysql_password`
  * `backup_mysql_host`
  * `backup_mysql_user`


**Optional**

Database to backup if is not set the playbook will backup all databases

  * `backup_mysql_database`

Override mysqldump options

  *  backup_mysql_options: "--single-transaction --routines --events --triggers"

Enable the validate task callback to check if the role setup succeded

    validate_task: true

Backup cron for mysql db dump

    backup_mysql_cron:
      name: snapshot
      job: "/usr/bin/cronlock ...
      minute: "0"
      hour: "3"
      day: "*"
      month: "*"
      weekday: "*"


Configure directory on the machine and s3 for the backup. Local backup is deleted after the upload in s3.

    backup_mysql_local_path: /home/admin/backups/mysql
    backup_mysql_s3_path: mysql

Configure the mysql backup script commands :

    backup_mysql_cmd: mysqldump --host={{ backup_mysql_host }} --user {{ backup_mysql_user }} --password={{ backup_mysql_password }} {{ backup_mysql_options }} {{ backup_mysql_database | default('--all-databases') }} > {{ backup_mysql_local_path }}/backup.sql


Configure backup retention handled by s3 bucket lifecycle policy : http://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-lifecycle.html

    # Example one : Every file in the s3 bucket will be removed after 365 days.
    backup_s3_expire_policy:
      permanently_delete: 365
      glacier_transition: 0

    # Example two : Every file myprefix* will be send on glacier after 60 days. And deleted from glacier after 365 days.
    backup_s3_expire_policy:
      prefix: "myprefix"
      permanently_delete: 365
      glacier_transition: 60



**Example :**

```
    - role: ansible-backup
      validate_task: true # Run the validator
      backup_type: mysql
      backup_mysql_bucket_name: "{{ backup_mysql_bucket_name }}"
      backup_mysql_password: password
      backup_mysql_host: 1.2.3.4
      backup_mysql_user: foo
```


Tests
=====

All backup types are self tested. After run the role, you can specify to automatically call the validate task callback with the `validate_task` variable.

To simplify the task, we are using kitchen.ci to test our ansible roles. Our kitchen test configuration use environment variable to allow the CI to configure the ansible playbook.

For the elasticsearch s3 backup, you have to provide all bucket informations to be able to test it :

    export AWS_ACCESS_KEY_ID=...
    export AWS_SECRET_ACCESS_KEY=...
    export AWS_DEFAULT_REGION=...
    export S3_BUCKET_NAME=...

Note : You also can add extra ansible command  args with `ANSIBLE_EXTRA_FLAGS` environment variable.


License
=======

cycloid.io


Migrate this module on ansible 2 notes
=======================================

**1)** Because of ansible `1.9.4` we removed the usage of `s3_lifecycle` module. The drawback is that we use aws cli with a template `s3/lifecycle.json.j2`. Aws cli don't do
incremental changes. It remove all existing policy on the bucket to apply them.

To migrate to `s3_lifecycle` module, your commit should looks like to https://bitbucket.org/cycloid-io/ansible-backup/commits/536c18fd4a3e2ef1c58183de4728da39708b8d36?at=master

**2)** The second point is to use `combine` on array. Combine is a way to merge 2 dict. I use it to have default dict value and be able to override only one field.
To add this behavior you could have a look of these commits : `git diff c3c2668...fea7d70`

The idea of `combine` behavior : 

  * variable `foo` in defaults. Foo is the array or part of the array a user want to override.
  * variable `default_foo` in vars directory is the defined role default variable.
  * variable `_foo` this variable is used inside the role. It result of `combine` between `foo` and `default_foo`

**3)** Remove the usage of `ansible_template_path_legacy` due to a bug with ansible `1.9.4`.
