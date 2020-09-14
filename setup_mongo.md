

helm repo add bitnami https://charts.bitnami.com/bitnami


kubectl create -f mongo-cert.yaml



helm install mongodb -f mongodb.yaml bitnami/mongodb

To get the root password run:

```bash
export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace default mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)
```

To connect to your database, create a MongoDB client container:

```bash
kubectl run --namespace default mongodb-client --rm --tty -i --restart='Never' --image docker.io/bitnami/mongodb:4.4 --command -- bash
```


If you need to upgrade mongo after deployment then the following commands can be used:

```bash
export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace default mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)
export MONGODB_REPLICA_SET_KEY=$(kubectl get secret --namespace default mongodb -o jsonpath="{.data.mongodb-replica-set-key}" | base64 --decode)
helm upgrade mongodb -f mongodb.yaml bitnami/mongodb --set auth.rootPassword=$MONGODB_ROOT_PASSWORD --set auth.replicaSetKey=$MONGODB_REPLICA_SET_KEY
```

