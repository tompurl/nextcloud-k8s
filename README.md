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
