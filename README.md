# Elasticsearch cluster and Kibana deployment

The aim of this repo is deploying an Elasticsearch cluster (3 nodes) with Kibana for testing.

## Docker-Compose deployment

Inside /docker folder, a docker-compose.yaml file can be founded where the cluster is written as code using v7.9.0:

Using this docker-compose file and executing the following line, the cluster is running:
```
$ sudo docker-compose up -d
```

http://localhost:5601/ -> Kibana interface.
http://localhost:9200/ -> Elasticsearch endopoint.

After testing these endpoints and check health status, It is possible to create an index via API and test its status, apart from populate it. Index accounts created and populated with a person with ID 1 and some info:
```
$ PUT http://localhost:9200/person
$ POST localhost:9200/accounts/person/1 
{
    "name" : "John",
    "lastname" : "Doe",
    "job_description" : "Systems administrator and Linux specialit"
}

$ GET localhost:9200/accounts/person/1
{
    "_index": "accounts",
    "_type": "person",
    "_id": "1",
    "_version": 1,
    "_seq_no": 1,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "name": "John",
        "lastname": "Doe",
        "job_description": "Systems administrator and Linux specialit"
    }
}

```

In order to test index is working fine, Kibana shows it via http://localhost:5601/app/management/data/index_management/indices.


## K8s deployment

Inside /k8s folder, some yaml files are written.
Before executing all these yaml files using Kubectl I had to restart my Minkube machine in order to downgrade Kubernetes version running inside it to v1.16 instead of 1.18. This is a workaround to use ELK on Minikube according to this [bug](https://github.com/kubernetes/minikube/issues/3869).

After that, I have created several files to complete the installation. In Elasticsearch case, these steps are needed to create a new namespace to work, create a service and create a StatefulSet for Elasticsearch cluster:

```
$ kubectl create -f namespace.yaml 
$ kubectl create -f elasticsearch-svc.yaml 
$ kubectl create -f elasticsearch-stfst.yaml 
$ kubectl rollout status sts/es-cluster --namespace=kube-logging
partitioned roll out complete: 3 new pods have been updated...
$ kubectl get pods -A -o wide
NAMESPACE      NAME                               READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
kube-logging   es-cluster-0                       1/1     Running   0          12m   172.17.0.3       minikube   <none>           <none>
kube-logging   es-cluster-1                       1/1     Running   0          11m   172.17.0.4       minikube   <none>           <none>
kube-logging   es-cluster-2                       1/1     Running   0          11m   172.17.0.5       minikube   <none>           <none>
```
To test if cluster is running properly and all nodes are recognized:

```
$ kubectl port-forward es-cluster-0 9200:9200 --namespace=kube-logging
$ curl http://localhost:9200/_cluster/state?pretty

  "nodes" : {
    "uAwZuKwBTRu8IrwYVsgZ5A" : {
      "name" : "es-cluster-0",
      "ephemeral_id" : "QI20NNEQSZ-bIQEt31V_lA",
      "transport_address" : "172.17.0.3:9300",
      "attributes" : {
        "ml.machine_memory" : "12585230336",
        "xpack.installed" : "true",
        "ml.max_open_jobs" : "20"
      }
    },
    "3bG0tvauSJKLOgdFLiEsJw" : {
      "name" : "es-cluster-1",
      "ephemeral_id" : "NV4C0nj8Tz-JIVTHddBrQw",
      "transport_address" : "172.17.0.4:9300",
      "attributes" : {
        "ml.machine_memory" : "12585230336",
        "ml.max_open_jobs" : "20",
        "xpack.installed" : "true"
      }
    },
    "2dvVOVOWRRihWJTSTzm9dw" : {
      "name" : "es-cluster-2",
      "ephemeral_id" : "V4stZnm4RViRUipVDh2VxQ",
      "transport_address" : "172.17.0.5:9300",
      "attributes" : {
        "ml.machine_memory" : "12585230336",
        "ml.max_open_jobs" : "20",
        "xpack.installed" : "true"
      }
    }
  },

```

Once Elasticsearch is running, Kibana needs to be started:

```
$ kubectl create -f kibana.yaml 
$ kubectl rollout status deployment/kibana --namespace=kube-logging
$ kubectl get pods --namespace=kube-logging
$ kubectl port-forward {kibana-pod-id} 5601:5601--namespace=kube-logging
$ kubectl get pods --namespace=kube-logging
NAME                      READY   STATUS    RESTARTS   AGE
es-cluster-0              1/1     Running   0          4m5s
es-cluster-1              1/1     Running   0          3m12s
es-cluster-2              1/1     Running   0          2m59s
kibana-69f959f8f9-t9rxk   1/1     Running   0          2m8s

```

After these pods are running and endpoints are ready, the final deployment is ready to be used as happened with Docker. Same steps to create and populate index are succesfully executed.