# homework otus mongo and kubernetes
Создаем кластер кубера:
```
C:\Users\1\AppData\Local\Google\Cloud SDK>gcloud beta container --project "mongo2021-19920819" clusters create "cluster-1" --zone "us-central1-c" --no-enable-basic-auth --cluster-version "1.21.6-gke.1500" --release-channel "regular" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "10" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --max-pods-per-node "110" --preemptible --num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/mongo2021-19920819/global/networks/default" --subnetwork "projects/mongo2021-19920819/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes --node-locations "us-central1-c"
Creating cluster cluster-1 in us-central1-c... Cluster is being health-checked (master is healthy)...done.
Created [https://container.googleapis.com/v1beta1/projects/mongo2021-19920819/zones/us-central1-c/clusters/cluster-1].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-central1-c/cluster-1?project=mongo2021-19920819
kubeconfig entry generated for cluster-1.
NAME       LOCATION       MASTER_VERSION   MASTER_IP       MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
cluster-1  us-central1-c  1.21.6-gke.1500  35.222.119.183  e2-medium     1.21.6-gke.1500  3          RUNNING
```

Прверим что все ноды в кластере успешно поднялсиь и кластер готов к работе:
```
C:\Users\1\AppData\Local\Google\Cloud SDK>kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.8.0.1     <none>        443/TCP   14m

C:\Users\1\AppData\Local\Google\Cloud SDK>kubectl get nodes
NAME                                       STATUS   ROLES    AGE   VERSION
gke-cluster-1-default-pool-1f443961-08kw   Ready    <none>   14m   v1.21.6-gke.1500
gke-cluster-1-default-pool-1f443961-c0l1   Ready    <none>   14m   v1.21.6-gke.1500
gke-cluster-1-default-pool-1f443961-wn5b   Ready    <none>   14m   v1.21.6-gke.1500
```

Добавляем репо от bitnami
```
C:\Users\1\AppData\Local\Google\Cloud SDK>helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
```

Правим конфиг с перемеными и подимаем mongodb:
```
C:\Users\1\AppData\Local\Google\Cloud SDK>helm upgrade --install mongo bitnami/mongodb --values C:\Users\1\Desktop\otus_mongo\otus_mongo\16_mongo_and_kuberntetes\values.yaml
Release "mongo" has been upgraded. Happy Helming!
NAME: mongo
LAST DEPLOYED: Sun Feb 27 23:10:29 2022
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: mongodb
CHART VERSION: 11.0.5
APP VERSION: 4.4.12

** Please be patient while the chart is being deployed **

MongoDB&reg; can be accessed on the following DNS name(s) and ports from within your cluster:

    mongo-mongodb-0.mongo-mongodb-headless.default.svc.cluster.local:27017
    mongo-mongodb-1.mongo-mongodb-headless.default.svc.cluster.local:27017
    mongo-mongodb-2.mongo-mongodb-headless.default.svc.cluster.local:27017

To get the root password run:

    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace default mongo-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)

To get the password for "mongo_otus" run:

    export MONGODB_PASSWORD=$(kubectl get secret --namespace default mongo-mongodb -o jsonpath="{.data.mongodb-passwords}" | base64 --decode | awk -F',' '{print $1}')

To connect to your database, create a MongoDB&reg; client container:

    kubectl run --namespace default mongo-mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:4.4.12-debian-10-r35 --command -- bash

Then, run the following command:
    mongo admin --host "mongo-mongodb-0.mongo-mongodb-headless.default.svc.cluster.local:27017,mongo-mongodb-1.mongo-mongodb-headless.default.svc.cluster.local:27017,mongo-mongodb-2.mongo-mongodb-headless.default.svc.cluster.local:27017" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD

To access the MongoDB&reg; Prometheus metrics, get the MongoDB&reg; Prometheus URL by running:

    kubectl port-forward --namespace default svc/mongo-mongodb-metrics 9216:9216 &
    echo "Prometheus Metrics URL: http://127.0.0.1:9216/metrics"

Then, open the obtained URL in a browser
```

Проверим по какому ip адресу и на каком порту работает mongo для того чтобы открыть внешний порт:
```
C:\Users\1\AppData\Local\Google\Cloud SDK>kubectl get all -n mongo
NAME                      TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)           AGE
service/mongodb-primary   NodePort   10.8.8.28    <none>        30001:30001/TCP   5m27s

C:\Users\1\AppData\Local\Google\Cloud SDK>gcloud compute firewall-rules create mongo --allow tcp:30001
Creating firewall...⠹Created [https://www.googleapis.com/compute/v1/projects/mongo2021-19920819/global/firewalls/mongo].
Creating firewall...done.
NAME   NETWORK  DIRECTION  PRIORITY  ALLOW      DENY  DISABLED
mongo  default  INGRESS    1000      tcp:30001        False
```

Порт открыли