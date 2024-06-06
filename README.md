# cassandra-backup-restore

## Install Cert Manager if it does not exist
```
helm repo add jetstack https://charts.jetstack.io

helm repo update
```

```
helm install cert-manager jetstack/cert-manager \
     --namespace cert-manager --create-namespace --set installCRDs=true
```

### Install Cassandra using the K8sCassandra Operator 
```
helm repo add k8ssandra https://helm.k8ssandra.io/stable
helm repo update
```
```
helm install k8ssandra-operator k8ssandra/k8ssandra-operator -n k8ssandra-operator --create-namespace
```
## If you want to use the operator more globally install it with the global scope enabled
```
helm install k8ssandra-operator k8ssandra/k8ssandra-operator -n k8ssandra-operator \ 
     --set global.clusterScoped=true --create-namespace
```

## Check to see the operator was installed 
```
kubectl get pods -n k8ssandra-operator
```

## Deploy a single node Cassandra Cluster in the Operator Namespace

kubectl apply -f cassandra_with_medusa.yaml


## Load some data 


### Retreive database Credentials 

kubectl get secret -n k8ssandra-operator demo-superuser -ojsonpath='{.data.username}' | base64 -d

kubectl get secret -n k8ssandra-operator demo-superuser -ojsonpath='{.data.password}' | base64 -d

### Shell into the Cassandra Node once up

```
kubectl exec -it -n k8ssandra-operator -c cassandra demo-dc1-default-sts-0  -- bash
```


### Activate the CQLSH Shell

```
cqlsh -u demo-superuser -p pwd
```

### Once shelled in, create keyspace and load data

#### Create keyspace
```
 CREATE KEYSPACE united_states WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
   
USE united_states;
```
#### Create Table
```
CREATE TABLE cities_by_state ( state text,  name text,population int,PRIMARY KEY ((state), name) );
```

#### Load data 
```
INSERT INTO cities_by_state (state, name, population) VALUES ('New York','New York City',8622357);
INSERT INTO cities_by_state (state, name, population) VALUES ('California','Los Angeles',4085014);
INSERT INTO cities_by_state (state, name, population)  VALUES ('Illinois','Chicago',2670406);
INSERT INTO cities_by_state (state, name, population) VALUES ('Texas','Houston',2378146);
INSERT INTO cities_by_state (state, name, population) VALUES ('Arizona','Phoenix',1743469);
INSERT INTO cities_by_state (state, name, population) VALUES ('Pennsylvania','Philadelphia',1590402);
INSERT INTO cities_by_state (state, name, population) VALUES ('Texas','San Antonio',1579504);
INSERT INTO cities_by_state (state, name, population) VALUES ('Texas','Dallas',1400337);
INSERT INTO cities_by_state (state, name, population) VALUES ('California','San Jose',1036242);
```
#### Validate Data
```
SELECT * FROM cities_by_state;
```


## Initialize a backup 

```
kubectl apply -f backup.yaml

```

## Wait until backup is finished 

```
kubectl get MedusaBackupJob -n k8ssandra-operator -w
```

Once the finish Column is set, you can describe the backup to see details 

```
kubeclt get MedusaBackupJob -n k8ssandra-operator medusa-backup1 -oyaml
```


After validating backup job got all your nodes, in our case a single node, we will drop the keyspace that contains our sample data

## Delete data that was loaded for testing 

### Shell into the Cassandra Node once up

```
kubectl exec -it -n k8ssandra-operator -c cassandra demo-dc1-default-sts-0  -- bash
```


### Activate the CQLSH Shell

```
cqlsh -u demo-superuser -p pwd
```

### Drop the keyspace

```
drop keyspace united_states; 

```
### Validate

```
describe keyspaces

```

## Resotration 

Now that we have lost our data, let us restore it 

```
kubectl apply -f restore.yaml
```

until restore is finished 

```
kubectl get MedusaRestoreJob -n k8ssandra-operator -w
```

Once the finish Column is set, ensure your cluster node is up and running.


```
kubectl get pods -n k8ssandra-operator 
```


## Validate your data was restored 



### Shell into the Cassandra Node once up

```
kubectl exec -it -n k8ssandra-operator -c cassandra demo-dc1-default-sts-0  -- bash
```


### Activate the CQLSH Shell

```
cqlsh -u demo-superuser -p pwd
```

### Once shelled in, create keyspace and load data

#### Use keyspace
```   
USE united_states;
```
#### Query Table

#### Validate Data
```
SELECT * FROM cities_by_state;
```