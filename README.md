# Nextcloud Deployment For Kubernetes

## Overview

This is my attempt at creating my own "recipe" for deploying a Nextcloud instance
on top of Kubernetes. I decided to roll my own version of this because I was
running K3S on top of a cluster of Raspberry Pi's and the current Helm
charts don't support that setup very well.

The version of the scripts in the root directory are designed to run on top of a K3S
cluster running on Raspberry PI's. The scripts in the `do-version` folder are
designed to work on a Digital Ocean K8S cluster.

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

You also need a running Kubernetes cluster with which you can interact using
`kubectl` and `helm3`.

## Deploying The System
### Pre-Work
#### TLS
First you need to install `cert-manager` so we can use TLS:

``` console
# Make sure you're using the latest version
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager --version v0.15.1 --namespace cert-manager jetstack/cert-manager
```

Next we need to setup the `ClusterIssuer` and that depends on whether you're using a
self-signed cert or a Let's Encrypt cert. If it's self signed cert then run this:

``` console
kubectl apply -f self-signed-issuer.yaml
# Then manually create your cert like this:
kubectl apply -f nextcloud-cert.yaml
```

For Let's Encrypt you need to run this instead:

``` console
kubectl apply -f prod-issuer.yaml
# The cert is created automatically when you create your ingress
```

#### Load Balancer
##### K3S
Nothing yet, still working on this
##### DO
If your K8S cluster is new then you need to install the **Nginx Ingress Controller** like this:

``` console
helm install nginx-ingress stable/nginx-ingress --set controller.publishService.enabled=true
```
#### DNS
After creating your load balancer (which is done automatically when you install the
Nginx Ingress controller) get it's public IP address like this:

``` console
kubectl get svc nginx-ingress-controller
```

Then use your DNS provider to map some sort of A record to that. For me it's
`pf-do-lb.tompurl.com`. I then pointed a CNAME record of `docs.tompurl.com` at that.

### Base Installation
#### K3S
TODO

#### DO

``` console
kubectl apply -k ./
kubectl apply -f ./nextcloud-ingress.yml
```

## Testing

Visit https://docs.tompurl.com

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

## Deleting Everything

``` console
kubectl delete deployment mysql nextcloud-deployment
# Delete your PVC's - CAUTION - This will also delete your PV's
# Delete your services
# Delete your ingress
```

