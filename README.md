# Kubernetes Backup Demo

## Intro

Slides regarding this demo can be found [here](https://docs.google.com/presentation/d/1CcoOmlUJfZBU4AaICWjGjTtATMguI8IbCyAPZgFL2ek/edit#slide=id.g12481fe0f2f_0_7)
Reference links where I took ideas to create this:
- Velero - https://docs.bitnami.com/tutorials/backup-restore-data-mongodb-kubernetes/
- Kanister - https://github.com/kanisterio/kanister/blob/master/examples/mongodb/README.md

## Requirements

Define your environment variables with the appropriate values for your use case
```
export mongodb_namespace=mongo-demo
export stateless_namespace=demo
export mongodb_root_password=password
export mongodb_root_replica_set_key=xyz123QWE

# --from-file absolute path only, relative path will fail
export velero_service_account_key_file_path='velero-backups-service-acct.json'
export kanister_service_account_key_file_path='kanister-backups-service-acct.json'

# don't use the gs:// prefix
export backup_storage_bucket_name="my-gcs-bucket-name-change-me"
export gcp_project_id="gpc-project-id-change-me"
```

## Demo

### Stateless Deployment
```
kubectl get ns
kubectl create ns $stateless_namespace

kubectl apply -f k8s-example-manifests/ -n $stateless_namespace
kubectl get all -n $stateless_namespace
```

```
kubectl -n $stateless_namespace port-forward kuard 8080:8080
kubectl -n $stateless_namespace port-forward svc/example-service 8080:8080

open http://localhost:8080/
curl http://localhost:8080/
```

### MongoDB Deployment (Stateful)
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

```
helm install mongodb bitnami/mongodb \
  --namespace $mongodb_namespace \
  --set architecture=replicaset \
  --set auth.rootPassword=$mongodb_root_password \
  --set auth.replicaSetKey=$mongodb_root_replica_set_key \
  --version 10.10.2 --create-namespace

export MONGO_REPLICA_SET_KEY=$(kubectl -n $mongodb_namespace get secret mongodb -o jsonpath="{.data.mongodb-replica-set-key}" | base64 --decode)
```

```
kubectl -n $mongodb_namespace exec -it mongodb-0 -- bash
mongo --host 127.0.0.1 --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
```

:warning: If you get authentication errors it might be because of the attached persistent volumes.
Old password remain in that storage so when db starts it would load that password from storage not the new provided.

Create some data entries
```
mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.insert({'name' : 'Nicks', 'cuisine' : 'Greek', 'id' : '8675309'})"
mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.insert({'name' : 'Luigi', 'cuisine' : 'Italian', 'id' : '8675310'})"
mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.insert({'name' : 'Raul', 'cuisine' : 'Cuban', 'id' : '8675311'})"
mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.insert({'name' : 'Jacks', 'cuisine' : 'American', 'id' : '8675312'})"

mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.find()"
```

### Velero

Requirements
```
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
brew install velero
```

Install Velero and the gcp/gcs Service Account cloud secret
```
kubectl create ns velero

kubectl create secret generic cloud-credentials --namespace velero --from-file=cloud=$velero_service_account_key_file_path

helm install velero vmware-tanzu/velero  --namespace velero -f  ~/src/nicolas-g/wiki/kubernetes/velero/velero-gcp-values.yaml --version 2.29.0
```

#### Velero Backup cluster namespace
```
kubectl get all -n $stateless_namespace
velero backup create ${stateless_namespace}-backup --include-namespaces $stateless_namespace --exclude-resources secrets
velero backup describe ${stateless_namespace}-backup  --details
kubectl get backups -n velero
```

#### Velero Backup MongoDB
```
kubectl get po -n $mongodb_namespace -l app.kubernetes.io/name=mongodb
velero backup create mongo-backup --include-namespaces $mongodb_namespace --include-resources pvc,pv --selector app.kubernetes.io/name=mongodb
velero backup describe mongo-backup  --details
kubectl get backups -n velero
```

#### Simulate a disaster
Delete all resources in a namespace
```
kubectl delete namespace ${mongodb_namespace}
kubectl delete namespace ${stateless_namespace}
```

Destroy a kubernetes cluster or delete DB data
```
mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.find()"
mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.drop()"
mongo admin --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --quiet --eval "db.restaurants.find()"
```


#### Velero Restore namespace
```
velero create restore ${stateless_namespace}-restore --from-backup ${stateless_namespace}-backup
velero restore describe ${stateless_namespace}-restore
kubectl -n $stateless_namespace get all
```

#### Velero Restore Mongo
```
velero create restore mongo-backup --from-backup mongo-backup

velero restore describe mongo-backup
kubectl -n velero get restore
kubectl -n $mongodb_namespace get pvc


# !!! delete delete all the secondary volumes (SECONDARY-PVC-NAMES)
kubectl -n $mongodb_namespace delete pvc datadir-mongodb-1
kubectl -n $mongodb_namespace get pvc

kubectl get pv
kubectl delete pvc ${mongodb_secondary_pv}
```

WARNING:
  It is important to create the new deployment on the destination cluster using the same namespace, deployment name,
  credentials and replicaset key as the original deployment on the source cluster.

Remember to use --set persistence.existingClaim=PRIMARY-PVC-NAME
```
helm install mongodb bitnami/mongodb \
  --namespace $mongodb_namespace \
  --set architecture=replicaset \
  --set auth.rootPassword=$mongodb_root_password \
  --set auth.replicaSetKey=$mongodb_root_replica_set_key \
  --set persistence.existingClaim=datadir-mongodb-0 \
  --version 10.10.2 --create-namespace

# --set auth.replicaSetKey=$MONGO_REPLICA_SET_KEY \
```

#### Velero Schedule backups
```
velero create schedule mongodbchedule --schedule="@every 1m" --include-namespaces $mongodb_namespace --include-resources pvc,pv
```

### Kanister

Install Kanister
```
helm install kanister --namespace kanister kanister/kanister-operator --create-namespace --version 0.74.0
```

#### Kanister profile
```
helm install kanister-gcp-profile kanister/profile --set defaultProfile=false --set profileName=gcs --set location.type='gcs' --set gcp.projectID="${gcp_project_id}" --set-file gcp.serviceKey=$kanister_service_account_key_file_path --set location.bucket=$backup_storage_bucket_name --set location.prefix='kanister' --namespace $mongodb_namespace --version 0.74.0

# running kanctl will perform SA permissions check
# kanctl create profile gcp --bucket $backup_storage_bucket_name --region us-east1 --project-id us-con-gcp-npr-0000186-081219 --service-key $kanister_service_account_key_file_path -n $mongodb_namespace --verbose
```

#### Kanister blueprint
```
kubectl apply -f https://raw.githubusercontent.com/kanisterio/kanister/master/examples/mongodb/mongo-blueprint.yaml -n kanister
kubectl -n kanister get blueprint mongodb-blueprint
```

#### Kanister backup Actionset
```
kanctl create actionset --action backup --namespace kanister --blueprint mongodb-blueprint --statefulset $mongodb_namespace/mongodb --profile $mongodb_namespace/gcs

-> actionset backup-fxfg5 created

kubectl -n kanister describe actionset backup-fxfg5
kubectl -n kanister logs -f -l app=kanister-operator
```

#### kanister restore actionset
```
# get the list of available backups
kubectl -n kanister get actionset
--> "backup-t2vhf"

kanctl --namespace kanister create actionset --action restore --from "backup-t2vhf"

# get the list of available restore
kubectl -n kanister get actionset
kubectl -n kanister describe actionset restore-backup-h2jzw-7mqll
```


# Clean

```
helm -n $mongodb_namespace uninstall mongodb

helm -n velero uninstall velero
helm -n kanister uninstall kanister
k delete -f k8s-example-manifests
k delete ns $stateless_namespace

k delete ns velero
k delete ns kanister

k delete pv (primary and secondary)
```
