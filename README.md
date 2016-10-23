# World Simplest BOSH Release

## Create a project

```
bosh init release simple-http-server-boshrelease --git
cd simple-http-server-boshrelease
```

## Create a job

```
bosh generate job simple-http-server
```

Create `jobs/simple-http-server/templates/ctl.erb`:

``` bash
#!/bin/bash

JOB_NAME=simple-http-server
RUN_DIR=/var/vcap/sys/run/$JOB_NAME
LOG_DIR=/var/vcap/sys/log/$JOB_NAME
PIDFILE=${RUN_DIR}/pid
PORT=<%= p("server.port") %>

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R vcap:vcap $RUN_DIR $LOG_DIR

    echo $$ > $PIDFILE
    exec chpst -u vcap:vcap python -m SimpleHTTPServer $PORT \
         >>$LOG_DIR/$JOB_NAME.log 2>&1

    ;;

  stop)
    kill -9 `cat $PIDFILE`
    rm -f $PIDFILE

    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;

esac
```

Create `jobs/simple-http-server/spec`:

``` yaml
---
name: simple-http-server
templates:
  ctl.erb: bin/ctl
packages:
```

Create `jobs/simple-http-server/monit`:

```
check process simple-http-server
  with pidfile /var/vcap/sys/run/simple-http-server/pid
  start program "/var/vcap/jobs/simple-http-server/bin/ctl start"
  stop program "/var/vcap/jobs/simple-http-server/bin/ctl stop"
  group vcap
```

## Create a package

```
bosh generate package simple-http-server
```

Create source files:

```
mkdir src/simple-http-server
echo 'Hello BOSH!' > src/simple-http-server/index.html
```

Create `packages/simple-http-server/packaging`:

```
# abort script on any command that exits with a non zero value
set -e -x

cp -a simple-http-server/index.html $BOSH_INSTALL_TARGET
```

Create `jobs/simple-http-server/spec`:

``` yaml
---
name: simple-http-server
templates:
  ctl.erb: bin/ctl
packages:
- simple-http-server
properties:
  server.port:
    description: "Port on which server is listening"
    default: 8080
```

## Create and Upload a release

Create `config/final.yml`:

```
---
blobstore:
  provider: local
  options:
    blobstore_path: /tmp/simple-http-server
final_name: simple-http-server
```

and

```
bosh create release --name simple-http-server --force && bosh upload release
```

## Deploy a cluster

```
mkdir manifest
```

Create `manifest/cloud-config-warden.yml` (if `bosh cloud-config` returns empty):

``` yaml
azs:
- name: z1

vm_types:
- name: default

disk_types:
- name: default
  disk_size: 1024

compilation:
  workers: 5
  az: z1
  reuse_compilation_vms: true
  vm_type: default
  network: default

networks:
- name: default
  type: manual
  subnets:
  - azs: [z1]
    range: 10.244.0.0/24
    reserved: [10.244.0.1]
    static:
    - 10.244.0.2 - 10.244.0.30
```

Create `manifest/simple-http-server.yml`:

``` yaml
---
name: simple-http-server

director_uuid: <%= `bosh status --uuid` %>

releases:
- name: simple-http-server
  version: latest

stemcells:
- os: ubuntu-trusty
  alias: ubuntu
  version: latest

instance_groups:
- name: simple-http-server
  jobs:
  - name: simple-http-server
    release: simple-http-server
    properties:
      server:
        port: 8080
  instances: 1
  stemcell: ubuntu
  azs: [z1]
  vm_type: default
  persistent_disk_type: default
  networks:
  - name: default
    static_ips: [10.244.0.10]

update:
  canaries: 1
  max_in_flight: 3
  canary_watch_time: 30000-600000
  update_watch_time: 5000-600000
```

Finally,

```
bosh update cloud-config manifest/cloud-config-warden.yml 
bosh deployment manifest/simple-http-server.yml
bosh -n deploy
```

You can see the deployed VM(s):

```
$ bosh vms
Acting as user 'admin' on 'Bosh Lite Director'
Deployment 'simple-http-server'

Director task 1192

Task 1192 done

+-------------------------------------------------------------+---------+----+---------+-------------+
| VM                                                          | State   | AZ | VM Type | IPs         |
+-------------------------------------------------------------+---------+----+---------+-------------+
| simple-http-server/0 (ebd213dc-9763-4d3a-add7-325dc1aec97a) | running | z1 | default | 10.244.0.10 |
+-------------------------------------------------------------+---------+----+---------+-------------+

VMs total: 1
```

Hit the http server:

```
$ curl 10.244.0.10:8080
Hello BOSH!
```
