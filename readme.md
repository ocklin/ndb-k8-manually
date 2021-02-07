# Demo 

Deploy MySQL NDB CLuster in Kubernetes manually. 

This will uses standard MySQL Cluster images from dockerhub.

# Deploy

## ConfigMaps, PersistentVolumes and Services

Create a ConfigMap resource named `ndb-configmap` containing the configuration.

```
$ kubectl create -f configmap.yaml 
```

Create the Service `ndb-svc`.  The `ndb-svc` Headless Service is used by the StatefulSets for the Management Servers and Data Nodes. This is necessary for the unique network identities.

The `service.yaml` file also contains the LoadBalancer Service `mgmd-ext-svc` in order to reach the Management Server from outside the Kubernetes cluster.

```
$ kubectl create  -f service.yaml 
```

Create the Persistent Volume Claim `ndb-pv-claim` as well used by both - the Management Server and Data Nodes. In StatefulSets they provide stable and persistent storage.

```
$ kubectl create -f pvc.yaml
```

## Start Management Server

Now we create the StatefulSet `mgmd` with Management Servers.

```
$ kubectl create -f mgmd-sfset.yaml
```

Now is a good time to check that the service is available:

```
$ kubectl get service/mgmd-ext-svc
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
mgmd-ext-svc   LoadBalancer   10.101.218.252   10.101.218.252   1186:30764/TCP   12m
```

We can use the External-IP to connect to the Management Server
and check cluster state. So far only the Management Server is 
started.

```
$ bin/ndb_mgm -c 10.101.218.252:1186 
-- NDB Cluster -- Management Client --
ndb_mgm> show
Connected to Management Server at: 10.101.218.252:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     1 node(s)
id=4 (not connected, accepting connect from ndb-0.ndb-svc.default.svc.cluster.local)

[ndb_mgmd(MGM)] 1 node(s)
id=1    @172.17.0.3  (mysql-8.0.22 ndb-8.0.22)

[mysqld(API)]   2 node(s)
id=5 (not connected, accepting connect from any host)
id=6 (not connected, accepting connect from any host)
```

Obviously also the shell in the Management Servers pod containers could be used.

```
$ kubectl exec -ti mgmd-0 -- /bin/bash
bash-4.2# ndb_mgm -c localhost:1186 
```

## Start Data Nodes

And now create the StatefulSet `ndb` with 2 pods running Data Nodes.

```
$ kubectl create -f ndb-sfset.yaml
```

## Start MySQL Server Deployment

The next step creates all Services, ConfigMaps and PersistenVolumeClaims 
for the MySQL Servers

```
$ kubectl create -f mysql-prep.yaml
```

Last but not least create a Deployment with MySQL Servers. 
From an NDB Cluster perspective they are stateless 
and do not require any stable network identity or storage.

```
$ kubectl create -f mysql-deployment.yaml
```

Thats all. Now a very simple 1 Data Node MySQL NDB Cluster is running and accessible through the MySQL Server SQL frontend.

## Using cluster through the MySQL Server 

Lets do that from the MySQL Server pod as we are not granted access to the MySQL 
Server from our external IP yet.

```
$ kubectl get pods 
NAME                    READY   STATUS    RESTARTS   AGE
mgmd-0                  1/1     Running   0          26m
mysql-675dc6dbc-jj64t   2/2     Running   0          14m
ndb-0                   1/1     Running   0          18m
```

```
$ kubectl exec -ti mysql-675dc6dbc-jj64t -- /bin/bash
bash-4.2# mysql -u root
mysql> 
```

```
mysql> create database test;
mysql> use test
mysql> CREATE TABLE t1 (i int primary key) engine = ndb;
```

```
```

# Using with minikube

```
$ minkube start
```

Set environment to use docker in minikube:

```
$ eval $(minikube docker-env)         
```

Starting the minikube tunnel is important if you want to use 
LoadBalancer service to contact 
e.g. the Management Server or MySQL Servers from outside the
Kubernetes cluster:

```
$ minikube tunnel
```