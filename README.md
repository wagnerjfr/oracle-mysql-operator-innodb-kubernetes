# `oracle-mysql-operator-innodb-kubernetes`

Deploying MySQL High Available solution ([InnoDB Cluster](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-userguide.html) + [Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html)) inside Kubernetes using the **Oracle MySQL Operator**.

## References:
- [Introducing the Oracle MySQL Operator for Kubernetes](https://medium.com/oracledevs/introducing-the-oracle-mysql-operator-for-kubernetes-b06bd0608726)
- [Oracle MySQL Operator GitGub Tutorial](https://github.com/oracle/mysql-operator/blob/master/docs/tutorial.md)
- [Setting up MySQL InnoDB Cluster using Docker containers](https://github.com/wagnerjfr/mysql-innodb-cluster)
- [Manually install Kubernetes on your laptop computer in minutes](https://github.com/wagnerjfr/wagnerjfr-kubernetes-cluster-vm-vagrant):star:

## Prerequisites
- A Kubernetes v1.8.0+ cluster
  - **P.S** this project uses the above:arrow_up: mentioned Kubernetes [solution](https://github.com/wagnerjfr/wagnerjfr-kubernetes-cluster-vm-vagrant) running locally in the host computer
- [Helm](https://helm.sh/docs/intro/install/) installed and configured in your cluster.

## Oracle MySQL Operator
### 1.1. Git clone
Clone the project and `cd` into the folder:
```
$ git clone https://github.com/oracle/mysql-operator

$ cd mysql-operator
```

### 1.2. Create a namespace
First create a namespace for the mysql-operator:
```
$ kubectl create ns mysql-operator
```

### 1.3. Installing the Chart
The below command deploys the MySQL Operator on the Kubernetes cluster in the default configuration.
```
$ helm install mysql-operator mysql-operator
```
Output:
```console
NAME: mysql-operator
LAST DEPLOYED: Sat Dec 14 14:01:37 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thanks for installing the MySQL Operator.

Check if the operator is running with

kubectl -n mysql-operator get po
```
Check the install:
```console
$ kubectl -n mysql-operator get po

NAME                            READY   STATUS    RESTARTS   AGE
mysql-operator-65475db5-c95f2   1/1     Running   0          88s
```
*[Optional]* To list all of the releases, run:
```console
$ helm list

NAME          	NAMESPACE	REVISION	UPDATED                               	STATUS  	CHART               	APP VERSION
mysql-operator	default  	1       	2019-12-14 14:03:58.40078221 +0000 UTC	deployed	mysql-operator-0.2.1
```

## 2. Create a simple MySQL InnoDB Cluster

### 2.1. Setting up
The first time you create a MySQL Cluster in a namespace (other than in the namespace into which you installed the `mysql-operator`) you need to create the mysql-agent ServiceAccount and RoleBinding in that namespace, in our case will be `innodb-cluster`, run:
```
$ kubectl create namespace innodb-cluster
```
Then run:
```
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysql-agent
  namespace: innodb-cluster
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: mysql-agent
  namespace: innodb-cluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mysql-agent
subjects:
- kind: ServiceAccount
  name: mysql-agent
  namespace: innodb-cluster
EOF
```

Similar output must be displayed:
```console
serviceaccount/mysql-agent created
rolebinding.rbac.authorization.k8s.io/mysql-agent created
```

### 2.2. InnoDB Cluster
The next command will deploy a MySQL InnoDB Cluster with 4 members and single primary mode (`multiMaster: false`). 
- `Single Primary`: Just one group member *(primary)* is set to read-write mode. All the others *(secondaries)* are set to read-only mode.
- `Multi-Primary`: All group members are *primaries* and set to read-write mode and there is no need to engage an election.

To find out how to create larger clusters and configure storage see [clusters](https://github.com/oracle/mysql-operator/blob/master/docs/user/clusters.md#clusters).

Create your MySQL password with a password field. Mine is `secret`. *(P.S. The rest of the tutorial will use `secret` as password)*
```
$ kubectl create secret generic -n innodb-cluster mysql-root-user-secret --from-literal=password=secret
```

Create a new cluster:
```
$ cat <<EOF | kubectl create -f -
apiVersion: mysql.oracle.com/v1alpha1
kind: Cluster
metadata:
  name: my-app-db
  namespace: innodb-cluster
spec:
  multiMaster: false
  members: 4
  rootPasswordSecret:
    name: mysql-root-user-secret
EOF
```
Output:
```console
cluster.mysql.oracle.com/my-app-db created
```

To check whether it's deployed:
```console
$ kubectl -n innodb-cluster get mysqlclusters

NAME        AGE
my-app-db   1m
```

It will take some minutes till you get all pods running:
```console
$ kubectl -n innodb-cluster get pods

NAME           READY   STATUS              RESTARTS   AGE
my-app-db-0    2/2     Running             0          8m56s
my-app-db-1    2/2     Running             0          7m12s
my-app-db-2    2/2     Running             0          3m14s
my-app-db-3    2/2     Running             0          2m13s
```

*[Optional]* List all pods in the namespace, with more details
```
$ kubectl get pods -o wide -n innodb-cluster

NAME           READY   STATUS              RESTARTS   AGE     IP           NODE                 NOMINATED NODE
my-app-db-0    2/2     Running             0          8m58s   10.244.1.3   worker1.vagrant.vm   <none>
my-app-db-1    2/2     Running             0          7m14s   10.244.2.3   worker2.vagrant.vm   <none>
my-app-db-2    2/2     Running             0          3m16s   10.244.1.4   worker1.vagrant.vm   <none>
my-app-db-3    2/2     Running             0          2m15s   10.244.2.5   worker2.vagrant.vm   <none>
```

### 2.3. Monitoring and validating the cluster
Streaming data from a pod:
```console
$ kubectl logs -f my-app-db-0 -n innodb-cluster mysql

+ base=1000
++ grep -o '[^-]*$'
++ cat /etc/hostname
+ index=0
++ expr 1000 + 0
+ /entrypoint.sh --server_id=1000 --datadir=/var/lib/mysql --user=mysql --gtid_mode=ON --log-bin --binlog_checksum=NONE --enforce_gtid_consistency=ON --log-slave-updates=ON --binlog-format=ROW --master-info-repository=TABLE --relay-log-info-repository=TABLE --transaction-write-set-extraction=XXHASH64 --relay-log=my-app-db-0-relay-bin --report-host=my-app-db-0.my-app-db --log-error-verbosity=3
[Entrypoint] MySQL Docker Image 8.0.12-1.1.7
[Entrypoint] Initializing database
2019-12-14T14:38:17.515523Z 0 [Note] [MY-010096] [Server] Ignoring --secure-file-priv value as server is running with --initialize(-insecure).
2019-12-14T14:38:17.515597Z 0 [Note] [MY-010949] [Server] Basedir set to /usr/.
2019-12-14T14:38:17.515611Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.12) initializing of server in progress as process 27
2019-12-14T14:38:17.517890Z 0 [Note] [MY-010458] [Server] --initialize specified on an existing data directory.
2019-12-14T14:38:17.519036Z 0 [Note] [MY-010163] [Server] No argument was provided to --log-bin, and --log-bin-index was not used; so replication may break when this MySQL server acts as a master and has his hostname changed!! Please use '--log-bin=my-app-db-0-bin' to avoid this problem.
2019-12-14T14:38:17.523244Z 0 [Note] [MY-012366] [InnoDB] InnoDB: Using Linux native AIO
2019-12-14T14:38:17.523555Z 0 [Note] [MY-010747] [Server] Plugin 'FEDERATED' is disabled.
2019-12-14T14:38:17.525070Z 1 [Note] [MY-012274] [InnoDB] InnoDB: The first innodb_system data file 'ibdata1' did not exist. A new tablespace will be created!
...
```
*[Optional]* You can also check the logs from `my-app-db-1`, `my-app-db-2` or `my-app-db-3`.

Let's use a MySQL client container to verify that you can connect to MySQL from within the Kubernetes cluster.
```
$ kubectl run mysql-client --image=mysql:8.0 -n innodb-cluster -it --rm --restart=Never \
    -- mysql -h my-app-db -uroot -psecret -e 'SHOW DATABASES'
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------------------------+
| Database                      |
+-------------------------------+
| information_schema            |
| mysql                         |
| mysql_innodb_cluster_metadata |
| performance_schema            |
| sys                           |
+-------------------------------+
pod "mysql-client" deleted
```

Let's use the `Performance Schema tables` to monitor `Group Replication`:
```
$ kubectl run mysql-client --image=mysql:8.0 -n innodb-cluster -it --rm --restart=Never \
    -- mysql -h my-app-db -uroot -psecret -e 'SELECT * FROM performance_schema.replication_group_members'
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST           | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+-------------+----------------+
| group_replication_applier | 55a19bcf-1eb8-11ea-9ec1-eee593fa0e7e | my-app-db-0.my-app-db |        3306 | ONLINE       | PRIMARY     | 8.0.12         |
| group_replication_applier | 6a92e636-1eb8-11ea-a52b-f6c9be9815ee | my-app-db-1.my-app-db |        3306 | ONLINE       | SECONDARY   | 8.0.12         |
| group_replication_applier | 8c560b8f-1eb8-11ea-a031-a6d528525d75 | my-app-db-2.my-app-db |        3306 | ONLINE       | SECONDARY   | 8.0.12         |
| group_replication_applier | cccc24a7-1eb8-11ea-97f0-862bcaacbbd8 | my-app-db-3.my-app-db |        3306 | ONLINE       | SECONDARY   | 8.0.12         |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+-------------+----------------+
pod "mysql-client" deleted
```

## 3. Clean up

### 3.1. Deleting mysql-operator
To uninstall/delete the mysql-operator deployment, run:
```
$ helm delete mysql-operator
```
Checking the pods:
```console
$ kubectl -n innodb-cluster get pods

NAME          READY   STATUS        RESTARTS   AGE
my-app-db-0   2/2     Terminating   0          21m
my-app-db-1   2/2     Terminating   0          19m
my-app-db-2   2/2     Terminating   0          15m
my-app-db-3   2/2     Terminating   0          14m
```

### 3.2. Deleting the namespace
To delete the created namespace, run:
```
$ kubectl delete ns innodb-cluster
```
