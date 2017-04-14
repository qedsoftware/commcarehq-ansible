= installation issues =

collecting all the issues we had while trying to install the software

your experience may vary - code changes, some of these are dated Nov2016.

== Revoke devs ability to login as ansible ==
it fails at `Revoke devs ability to login as ansible`
reason: thsy're trying to do some user management - have set of keys and such.
If we go with this, whole section probably must be skipped (we don't want any of htir devs on our server, I guess).

workaround for now:
```diff
diff --git a/ansible/roles/users/tasks/main.yml b/ansible/roles/users/tasks/main.yml
index ec956e2..36ffdcd 100644
--- a/ansible/roles/users/tasks/main.yml
+++ b/ansible/roles/users/tasks/main.yml
@@ -36,7 +36,7 @@
- name: Revoke devs ability to login as ansible
sudo: yes
authorized_key: user=ansible state=absent key="{{ lookup('file', 'vars/users/' ~ item.0 ~ '.pub')
- when: item.1.stat.exists
+ when: False
with_together:
- dev_users.absent
- absent_user_keys.results
```

== Couchdb fails to start, because Scripts try to bind to host inventory address ==
and on aws external address is different from local one.
(there's no interface with address 54.175.135.134 on ec2-54-175-135-134.compute-1.amazonaws.com)

workaround
(not sure about 127.0.0.1 here - maybe we should settle with 0.0.0.0 and listern on all interfaces?)
```
--- a/ansible/roles/couchdb/tasks/main.yml
+++ b/ansible/roles/couchdb/tasks/main.yml
@@ -77,5 +77,5 @@
- name: Add CouchDB databases
shell: "{{ item }}"
with_items:
- - curl -X PUT "http://{{ localsettings.COUCH_SERVER_ROOT }}/{{ localsettings.COUCH_DATABASE_NAME }}"
- - curl -X PUT "http://{{ localsettings.COUCH_SERVER_ROOT }}/_config/admins/{{ localsettings.COUCH_USERNAME }}" -d '"{{ localsettings.COUCH_PASSWORD }}"'
+ - curl -X PUT "http://127.0.0.1:5984/{{ localsettings.COUCH_DATABASE_NAME }}"
+ - curl -X PUT "http://127.0.0.1:5984/_config/admins/{{ localsettings.COUCH_USERNAME }}" -d '"{{ localsettings.COUCH_PASSWORD }}"'
diff --git a/ansible/roles/couchdb/templates/local.ini.j2 b/ansible/roles/couchdb/templates/local.ini.j2
index 05abafb..d74df15 100644
--- a/ansible/roles/couchdb/templates/local.ini.j2
+++ b/ansible/roles/couchdb/templates/local.ini.j2
@@ -13,7 +13,7 @@

[httpd]
port = 5984
-bind_address = {{ groups['couchdb'][0] }}
+bind_address = 127.0.0.1
; Options for the MochiWeb HTTP server.
;server_options = [{backlog, 128}, {acceptor_pool_size, 16}]
; For more socket options, consult Erlang's module 'inet' man page.
```
Copy Elasticsearch Config fails because they did't add S3 access vars
that is right, there's no AMAZON_S3_ACCESS_KEY in our var file under localsettings_private.

Lets add stub (I have no idea what key with what access to what data do we need, so thats best we can do now):
```
--- a/ansible/vars/dev/dev_public.yml
+++ b/ansible/vars/dev/dev_public.yml
@@ -142,7 +142,10 @@ postgresql_dbs:
# shards: [400, 511]
# - name: "{{localsettings.UCR_DATABASE_NAME}}"

+aws_region: 'stub'
localsettings:
+ AMAZON_S3_ACCESS_KEY: 'stub'
+ AMAZON_S3_SECRET_KEY: 'stub'
ALLOWED_HOSTS:
- localhost
- 127.0.0.1
```

More undefined variables: elasticsearch | Create initial snapshot
```
--- a/ansible/vars/dev/dev_public.yml
+++ b/ansible/vars/dev/dev_public.yml
@@ -6,6 +6,7 @@ NO_WWW_SITE_HOST: '{{ groups.proxy.0 }}'
J2ME_SITE_HOST: 'j2me.{{ groups.proxy.0 }}'

testing: True
+backup_es: False

etc_host_lines: []
# - "169.38.74.104 commcarehq-india.cloudant.com"

```



== could not open relation mapping file ==
```
TASK: [postgresql | Create PostgreSQL users] **********************************
failed: [192.168.33.16] => {"failed": true}
msg: unable to connect to database: FATAL:  could not open relation mapping file "global/pg_filenode.map": No such file or directory
```

Just retry, for me it passed this step fine (some PG did not finish
initialization?)


== Node is not reachable at Join nodes to cluster ==
```
TASK: [Join nodes to cluster] *************************************************
skipping: [192.168.33.16]
failed: [192.168.33.17] => {"changed": true, "cmd": ["riak-admin", "cluster", "join", "riak-192@192.168.33.16"], "delta": "0:00:00.210130", "end": "2017-02-01 14:25:06.832494", "rc": 1, "start": "2017-02-01 14:25:06.622364", "warnings": []}
stdout: Node riak-192@192.168.33.16 is not reachable!
```

For some reason after installation there is a default config file
that _prevents_ ansible from updating it (they put special chek for
this, if file exist they don't update it- probably their prod envs rely
on it).
Remove conffile it on all hosts, relaunch.
  /etc/riak/riak.conf
  /etc/riak-cs/riak-cs.conf
  /etc/stanchion/stanchion.conf


== Nginx fails to check its config because of missing certs ==

they have some `fake_ssl_cert: yes` clause
in their scripts, which omits copying REAL certs/keys,
but there's no sign of generating "fake" (self-signed)
certs - and at least nginx conf file requires that.

use self-signed ones on the host
```
for f in dev_nginx_commtrack dev_nginx
do
  cp -v /etc/ssl/certs/ssl-cert-snakeoil.pem /etc/pki/tls/certs/${f}_combined.crt
done
mkdir -p /etc/pki/tls/private
cp /etc/ssl/private/ssl-cert-snakeoil.key /etc/pki/tls/private/dev_nginx_commtrack.org.key
cp /etc/ssl/private/ssl-cert-snakeoil.key /etc/pki/tls/private/dev_nginx_commcarehq.org.key
```

== mysterious DimagiKeyStore ==

```
TASK: [keystore | Send Keystore] **********************************************
fatal: [192.168.33.15] => input file not found at /vagrant/ansible/roles/keystore/files/DimagiKeyStore or /vagrant/ansible/DimagiKeyStore
```

Readme mentions that:
The DimagiKeyStore file is not kept in source control.  When cloning this repo
the keystore files should be added to this directory.  Ansible is expecting it
to be named "DimagiKeyStore".

I have no idea what this keystore is, and nothing in this role gives any
clue (in fact, it only has 1 task in it: Send Keystore), so...

date > /vagrant/ansible/roles/keystore/files/DimagiKeyStore

== Vagrant boxes are to small to host npm ==

On initial run (while services are not running) `npm` works, but as playbook progresses
and kafka starts -> no more ram for npm to run, failure.
Easy one, just top Vagrant boxes 768Mb -> 2Gb.

== touchforms fails to create user ==

```
TASK: [touchforms | Touchforms user] ******************************************

  RAN: /home/cchq/www/dev/current/python_env/bin/python manage.py shell --plain

  STDOUT:


  STDERR:
Traceback (most recent call last):
  File "manage.py", line 113, in <module>
    patch_jsonfield()
  File "manage.py", line 73, in patch_jsonfield
    from django.core.exceptions import ValidationError
ImportError: No module named django.core.exceptions
```

Well, their dev enviromnet specifies nice var called `testing`,
which skips `pip install` steps. Remove it.

== undefined PG_DATABASE_PORT ==

```
TASK: [touchforms | Copy Jython localsettings.py] *****************************
fatal: [192.168.33.16] => {'msg': "AnsibleUndefinedVariable: One or more undefined variables: 'dict object' has no attribute 'PG_DATABASE_PORT'", 'failed': True}
fatal: [192.168.33.16] => {'msg': "AnsibleUndefinedVariable: One or more undefined variables: 'dict object' has no attribute 'PG_DATABASE_PORT'", 'failed': True}
```

`ansible/vars/dev/dev_public.yml` misses this PG_DATABASE_PORT var.
Added it.


== Missing variables, servers trying to bind to SERVER-ADDRESS ==

(on AWS external address is not bound to any of internal interfaces - services fail to start with "cannot bind" error):
too many occurences to list here, here's a commit:
https://github.com/qedsoftware/commcarehq-ansible/commit/03cf61ccfa23d90ddd027cb0459987e8a985fa63

== after installation site shows maintenance page ==

no django server is running after the installation.
theres supervisord to start, but its config files are not created.

Apparently theres _third_ repo (https://github.com/dimagi/commcare-hq-deploy) here that is briefly mentioned in second repo's README, and there are templates for
supervisord are there and should be generated manually (at least I found
zero mentions "main" playbook).

```
cd ~/www/dev/current
mkdir -p services/supervisor
for i in deployment/commcare-hq-deploy/fab/services/templates/*
do
  ./manage.py make_supervisor_conf --conf_file `basename $i` \
    --conf_destination services/supervisor --params \
      '{"project":"commcarehq",
        "environment": "prod",
        "code_current": "/home/cchq/www/dev/current",
        "django_bind": "127.0.0.1",
        "django_port": 9010,
        "flower_port": 5555,
        "log_dir":"/tmp",
        "virtualenv_current":"/home/cchq/www/dev/current/python_env",
        "sudo_user": "cchq",
        "celery_params": {"concurrency": 1}}'
done

service supervisord restart
```

== OfflineGenerationError fails at ./manage.py compress ==
Application still fails with
OfflineGenerationError: You have offline compression enabled but key "686a1a4dbebd595c60d75888683fb65e" is missing from offline manifest. You may need to run "python manage.py compress".
(collected from /home/cchq/www/dev/log/commcare.qed.ai-commcarehq.django.log)

and `./manage.py compress` that is supposed to fix it
fails with

```
Compressing... CommandError: An error occurred during rendering /home/cchq/www/dev/releases/2017-03-10_09.15/corehq/apps/hqadmin/templates/hqadmin/couch_changes.html:
FileError: '../../../../../../bower_components/bootstrap/less/normalize.less' wasn't found
in /home/cchq/www/dev/releases/2017-03-10_09.15/staticfiles/style/less/bootstrap.less on line 14, column 1:
```

No idea how to fix it, alas.
I lack experience in modern frontend.

== Docker installation fails ===
There are some docker images / instructions at
https://github.com/dimagi/commcare-hq/tree/master/docker

Step 1 (./scripts/docker runserver --bootstrap)
fails with
```
...
Waiting for services: COUCH:5984 POSTGRES:5432 REDIS:6379 KAFKA:9092 ELASTICSEARCH:9200 RIAKCS:9980
Waiting for TCP connection to COUCH @ 172.17.0.2:5984...COUCH ok
Waiting for TCP connection to POSTGRES @ 172.17.0.5:5432...POSTGRES ok
Waiting for TCP connection to REDIS @ 172.17.0.3:6379...REDIS ok
Waiting for TCP connection to KAFKA @ 172.17.0.7:9092...KAFKA ok
Waiting for TCP connection to ELASTICSEARCH @ 172.17.0.6:9200...ELASTICSEARCH ok
Waiting for TCP connection to RIAKCS @ 172.17.0.4:9980...RIAKCS ok
Services ready!
Traceback (most recent call last):
  File "./manage.py", line 118, in <module>
    execute_from_command_line(sys.argv)
  File "/vendor/lib/python2.7/site-packages/django/core/management/__init__.py", line 367, in execute_from_command_line
    utility.execute()
  File "/vendor/lib/python2.7/site-packages/django/core/management/__init__.py", line 341, in execute
    django.setup()
  File "/vendor/lib/python2.7/site-packages/django/__init__.py", line 22, in setup
    configure_logging(settings.LOGGING_CONFIG, settings.LOGGING)
  File "/vendor/lib/python2.7/site-packages/django/utils/log.py", line 75, in configure_logging
    logging_config_func(logging_settings)
  File "/usr/local/lib/python2.7/logging/config.py", line 794, in dictConfig
    dictConfigClass(config).configure()
  File "/usr/local/lib/python2.7/logging/config.py", line 559, in configure
    'filter %r: %s' % (name, e))
ValueError: Unable to configure filter 'hqcontext': Cannot resolve 'corehq.util.log.HQRequestFilter': No module named dimagi.utils.chunked
Exception KeyError: KeyError(139809728546288,) in <module 'threading' from '/usr/local/lib/python2.7/threading.pyc'> ignored

```
