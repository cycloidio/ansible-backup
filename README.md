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

Configure the retention policy of your backups :

    backup_es_retention:
      unit: days
      value: 30

Cron commands to backup and apply the policy retention

    backup_es_crons:
      - name: snapshot
        ...
        minute: "0"
        hour: "3"
      - name: retention command
        ...
        minute: "0"
        hour: "6"

Enable the validate task callback to check if the role setup succeded

    validate_task: true

**Example :**

```
  roles:
    - role: ansible-backup
      backup_type: elasticsearch
      backup_es_bucket_name: "{{ backup_es_bucket_name }}"
      backup_es_bucket_region: "{{ aws_default_region }}"
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

