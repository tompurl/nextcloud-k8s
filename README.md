# Nextcloud Deployment For Kubernetes

## Overview

This is my attempt at creating my own "recipe" for deploying a Nextcloud instance
on top of Kubernetes. I decided to roll my own version of this because I was
running K3S on top of a cluster of Raspberry Pi's and the current Helm
charts don't support that setup very well.

## Requriements

You need to include a ``kustomization.yaml`` file that looks something like
this:

``` yaml
secretGenerator:
- name: mysql-root-pass
  literals:
  - password=$omethingClever
- name: mysql-nc-user-pass
  literals:
  - password=@lsoSomethingClever
- name: nextcloud-admin-pass
  literals:
  - password=I@mSoSmart
resources:
- mysql-deployment.yaml
- nextcloud-deployment.yaml
```

## Deploying The System

``` console
kubectl apply -k ./
```

## Testing

TODO

## Deleting Everything

``` console
kubectl delete deployment mysql nextcloud-deployment
# Delete your PVC's
# Delete your services
```

## Post-Installation Setup
### Overview
These are the tasks that you would execute on a "real" system that you hope to maintain for the long-term.
### Pre-reqs
Some of these steps required `kubectl` plugins, which are installed and managed with `krew`. 

Here's the plugins that these steps require:

- [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/ "Installation steps for krew")
- [exec-as plugin](https://github.com/jordanwilson230/kubectl-plugins/tree/krew#kubectl-exec-as "Github page for the exec-as plugin")

### DB Perf Improvements
There are a couple of sql scripts that you're supposed to run that will help with performance significantly, and the sooner you run them the better :-) 

First let's run the `db:add-missing-indices` script like so:

``` bash
NC_POD=$(kubectl get pods | grep nextcloud | grep -vi mysql | grep -v svclb | awk '{print $1}')
kubectl exec-as -u www-data "$NC_POD" -- ./occ db:add-missing-indices
```

And then run the `occ db:convert-filecache-bigint` script. This script requires user
input so we need to open a bash prompt.

```
kubectl exec-as -u www-data "$NC_POD" -- /bin/bash

Connecting...
Pod: nextcloud-54847b5547-lppgj
Namespace: NONE
User: www-data
Container: NONE
Command: /bin/bash

www-data@nextcloud-54847b5547-lppgj:~/html$ ./occ db:convert-filecache-bigint
Following columns will be updated:

* mounts.storage_id
* mounts.root_id
* mounts.mount_id

This can take up to hours, depending on the number of files in your instance!
Continue with the conversion (y/n)? [n] y
```
Then simply exit when it's done. For me on a new instance it took about 5 seconds.


## Backups
### Everything Under The Data Folder

This includes basically all of your pictures and office docs and such, basically
everything that is hosted using webdav. Since we're pointing at an NFS partition you
just need to back that up.

### Everything Else Web-Related

This includes everything else under ``/var/www/html``. 

``` bash
NC_POD=$(kubectl get pods | grep nextcloud | grep -v mysql | grep -v svclb | awk '{print $1}')

kubectl cp default/$NC_POD:/var/www/html ./html-backup

# This backup contains the ``data`` dir, which we backup elsewhere
rm -rf ./html-backup/data

# Then zip it all up
tar cfz html-backup.$(date +%Y%m%d%H%M).tgz html-backup/

# Cleanup
rm -rf html-backup
```

### MySQL

``` bash
NC_SQL_POD=$(kubectl get pods | grep nextcloud-mysql | grep -v svclb | awk '{print $1}')
DB_PASS="thisIsCool"

(kubectl exec -it "$NC_SQL_POD" -- /usr/bin/mysqldump -u root --password=$DB_PASS nextcloud_db) > nextcloud_db.bkup.$(date +%Y%m%d%H%M).sql
```
