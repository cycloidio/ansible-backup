ansible-backup
==============

This role install and configure backups for several applications like elasticsearch and mongodb.


Dependances :

Credentials
https://github.com/elastic/elasticsearch-cloud-aws#generic-configuration

By default snapshot and retention command run with cronlock :
For cycloid cronlock is provided by ansible-base
If you don't want that, just override .......... commands


No actual retention policy https://github.com/elastic/elasticsearch/issues/3826
Will install elasticsearch-curator

Exemple :
    - role: aloysius.elasticsearch
      es_etc:
        cloud.aws.region: eu-west
        cloud.aws.access_key: "..."
        cloud.aws.secret_key: "..."

Need define :
TODO GET defaults/main.yml content



Generic role for software deployment

TODO le verify doit s'assurer qu'on peux se connecter au bucket
POST /_snapshot/my_backup/_verify


Test part :
--------------
$ export AWS_ACCESS_KEY_ID=...
$ export AWS_SECRET_ACCESS_KEY=...
$ export AWS_DEFAULT_REGION=...
$ export S3_BUCKET_NAME=...

Extra env var ANSIBLE_EXTRA_FLAGS

Self tested role :
  roles:
    - role: ansible-backup
      validate_task: true # Run only the validator                                                                                                                                                                                 




Requirements
------------

Nginx and supervisord if you enabled their flags

Role Variables
--------------
When true, stop supervisord of deployment_app_name at the beginning and restarted it at the end

    deployment_supervisord_enabled: false

When true, restart nginx at the end

    deployment_nginx_enabled: false


When true, stash sensu alert at the beginning end unstash them at the end 

    deployment_sensu_enabled: false

`deployment_sensu_stashes` sample:
    - body:
        path: "silence/CYCLOID-CI0-EU-WE1-PROD/nginx_proc_check"
        content:
          message: "deployment"
          source: "ansible"
        expire: 600

When true, sens a mail to `deployment_deploy_to: []`

    deployment_mail_enabled: false

License
-------

cycloid.io


###############


curl -XPUT 'localhost:9200/customer?pretty'
curl 'localhost:9200/_cat/indices?v'

curl -XPUT 'localhost:9200/customer/external/1?pretty' -d '
{
  "name": "John Doe"
}'
curl -XGET 'localhost:9200/customer/external/1?pretty'



# Create directory
curl -XPUT 'http://localhost:9200/_snapshot/my_s3_repository' -d '{
    "type": "s3",
    "settings": {
        "bucket": "gael-test",
        "region": "eu-west-1"
    }
}'

curl http://localhost:9200/_snapshot/s3_backup/ -v


PUT /_snapshot/s3_repository?verify=false
...

# create a dump
curl -XPUT  http://localhost:9200/_snapshot/my_s3_repository/snapshot_1?wait_for_completion=true



Once a snapshot is created information about this snapshot can be obtained using the following command:

GET /_snapshot/my_backup/snapshot_1
GET /_snapshot/my_backup/_all
curl http://localhost:9200/_snapshot/my_s3_repository/_all


  vars:
     local_home: "{{ lookup('env','HOME') }}"


export ANSIBLE_EXTRA_FLAGS="--tags ansible-backup"
#S3 policy
{
  "Id": "Policy1455543528849",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1455543516341",
      "Action": "s3:*",
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::gael-test",
      "Principal": {
        "AWS": [
          "661913936052"
        ]
      }
    }
  ]
}###
eu-west-1
apt-get install s3cmd
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=
export AWS_DEFAULT_REGION=eu-west-1

s3cmd ls s3://gael-test/


