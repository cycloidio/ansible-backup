ansible-backup
==============

This role install and configure backups for several applications like elasticsearch and mongodb.

No global requirements. It will depend of what type of backup you want. See specific Requirements section for each backup type.

The only things you need is to specify `backup_type` variable to define what kind of backup you want to setup.

elasticsearch
=============

For now on elasticsearch don't provide a "retention policy" feature for snapshots. So we are using the python module `elasticsearch-curator` to do the job (https://www.elastic.co/guide/en/elasticsearch/client/curator/index.html).

Exemple with `aloysius.elasticsearch` ansible role :

```
    - role: aloysius.elasticsearch
      es_etc:
        cloud.aws.region: "..."
        cloud.aws.access_key: "..."
        cloud.aws.secret_key: "..."
```

Requirements
------------

We are using `elasticsearch-cloud-aws` plugin to backup on s3 bucket. So ensure configured AWS credentials for this plugin (https://github.com/elastic/elasticsearch-cloud-aws#generic-configuration)

The default cron commands for snapshot and retention are using `cronlock`. For Cycloid `cronlock` is provided by the `ansible-base` role. If you don't want `cronlock`, just override the `backup_es_crons` variable.

Role Variables
--------------

**Required**

Name and region of the bucket to use for backups on s3

    backup_es_bucket_name: foo
    backup_es_bucket_region: us-east-1

**Optional**

Enable the validate task callback to check if the role setup succeded

    validate_task: true

Enable or not full backup (default false) :

    backup_es_backup_full: false


This role provide 2 type of backups for elasticsearch :

`FULL`  Play with elasticsearch behavior to be able to do full or incremental backups. For the full, we specify a new s3 directory inside the bucket each time.
That force elasticsearch to do a full snapshot.
You should use this mode if you want to let aws s3 bucket handle the retention or migrate data to glacier after a while.

Command to launch the backup : default increment

    backup_es_backup_cmd: ...
2 type de backups :
increment.
Regular incremental snapshot from elasticsearch
And need to launch the retention command


Full ou increment
Backup commande. Astuce pour full c'est la mm chose, juste changer

Rentention of snapshot in a directory.

Retention policy in s3.
You should know that elastic search do incremental backups.
If you want to delete files from s3 or automatically migrate them to glacier, you have one tricks.


The index snapshot process is incremental. In the process of making the index snapshot Elasticsearch analyses the list of the index files that are already stored in the repository and copies only files that were created or changed since the last snapshot.
When a snapshot is deleted from a repository, Elasticsearch deletes all files that are associated with the deleted snapshot and not used by any other snapshots. 

Elasticsearch do increment snapshot, refering to old one in the directory. Use the snapshot retention commande to
reduce the number of snapshot in a direcory current directory to keep efficient backup time.
https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html

By default this module create a new directory in the bucket each month (prefix-month number). So that means a new full backup each month

# Si le directory existe pas la 1er fois, ansible le crée pour valider le fonctionnement
# Script For recreate each backup the directory.
# Recreate syntaxe commande bash, on peux feinter avec $date pour créer un dossier par mois
if backup_es_retention, enable retention cron


Configure the retention policy of your backups :

    backup_es_retention:
      unit: days
      value: 30

Cron commands to backup and apply the policy retention

    backup_es_cron:
      - name: snapshot
        ...
        minute: "0"
        hour: "3"

ENSURE THE SSH USER HAVE S3 access to configure the policy
# s3 policy: enable or not if >0
# s3 directory name used in elastic-directory-create.sh
# Retention time inside directory. (call at every backups)
# Backup cron script
#   * call create script.sh
#   * call backup cmd
#   * if retention >0 exec retention







**Example :**

```
  roles:
    - role: ansible-backup
      backup_type: elasticsearch
      backup_es_bucket_name: "{{ backup_es_bucket_name }}"
      backup_es_bucket_region: "{{ aws_default_region }}"
```

Mongodb
=======

Based on https://docs.mongodb.org/v3.0/tutorial/backup-small-sharded-cluster-with-mongodump/
Don't use for a bug shared mongodb cluster.

Requirements
------------

To upload mongodb dump on s3 bucket, this role use `aws-cli`. It's important that you configured `aws-cli` for the user who will launch the cron backup command.
For example you can use a `.boto` file. (http://boto.cloudhackers.com/en/latest/boto_config_tut.html)

Role Variables
--------------

# References
https://docs.mongodb.org/v3.0/tutorial/backup-small-sharded-cluster-with-mongodump/
https://docs.mongodb.org/v3.0/administration/backup-sharded-clusters/
# Expire configuration
# doc http://docs.aws.amazon.com/cli/latest/reference/s3api/get-bucket-lifecycle.html
http://docs.ansible.com/ansible/intro_configuration.html#sudo-flags

**Required**

backup_mongo_bucket_name


**Optional**
    backup_mongo_cron:
      name: snapshot
      job: "/usr/bin/cronlock mongodump"
      minute: "0"
      hour: "3"
      day: "*"
      month: "*"
      weekday: "*"



Customize the bump
backup_mongo_with_lock default false
backup_mongo_lock_cmd: /usr/bin/mongo admin --eval "printjson(db.fsyncLock())"
backup_mongo_unlock_cmd: /usr/bin/mongo admin --eval "printjson(db.fsyncUnlock())"

backup_mongo_local_path: /opt/mongo-backups
backup_mongo_cmd: /usr/bin/mongodump --oplog --out {{ backup_mongo_local_path }}





**Example :**


TODO : validate task
  * Test that the cron user can access to the s3 registry

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

