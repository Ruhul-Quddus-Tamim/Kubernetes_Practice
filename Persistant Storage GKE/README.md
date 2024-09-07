# Objectives

- Create manifests for PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs) for Google Cloud persistent disks (dynamically created or existing).
- Mount Google Cloud persistent disk PVCs as volumes in Pods.
- Use manifests to create StatefulSets.
- Mount Google Cloud persistent disk PVCs as volumes in StatefulSets.

# Create PVs and PVCs

### Connect to the lab GKE cluster
<img width="1280" alt="Screenshot 2024-09-07 at 4 35 47 PM" src="https://github.com/user-attachments/assets/23c23c84-eaa1-4499-ac7b-94661041ca12">


1. In Cloud Shell, type the following command to set the environment variable for the zone and cluster name:
```
export my_region=Region
export my_cluster=autopilot-cluster-1
```

2. Configure tab completion for the kubectl command-line tool:
```
source <(kubectl completion bash)
```

3. Configure access to your cluster for kubectl:
```
gcloud container clusters get-credentials $my_cluster --region $my_region
```

### Create and apply a manifest with a PVC
Most of the time, you don't need to directly configure PV objects or create Compute Engine persistent disks. Instead, you can create a PVC, and Kubernetes automatically provisions a persistent disk for you.

Let's creates a 30 gigabyte PVC called `hello-web-disk` that can be mounted as a read-write volume on a single node at a time.

1. To create the PVC, execute the following command:
```
kubectl apply -f pvc-demo.yaml
```

2. To show your newly created PVC, execute the following command:
```
kubectl get persistentvolumeclaim
```

**Partial output:**
```
NAME           STATUS    VOLUME       CAPACITY   ACCESS MODES   STORAGE CLASS   AGE
hello-web-disk Pending                                          standard-rwo    15s
```

# Mount and verify Google Cloud persistent disk PVCs in Pods
 Attach your persistent disk PVC to a Pod. You mount the PVC as a volume as part of the manifest for the Pod.

### Mount the PVC to a Pod
Using manifest file `pod-volume-demo.yaml` to deploy an nginx container, attache the `pvc-demo-volume` to the Pod and mount that volume to the path `/var/www/html` inside the nginx container. Files saved to this directory inside the container will be saved to the persistent volume and persist even if the Pod and the container are shutdown and recreated.

1. To create the Pod with the volume, execute the following command:
```
kubectl apply -f pod-volume-demo.yaml
```

2. List the Pods in the cluster:
```
kubectl get pods
```

**Output:**
```
NAME          READY    STATUS              RESTARTS   AGE
pvc-demo-pod  0/1      ContainerCreating   0          18s
```

If you do this quickly after creating the Pod, you will see the status listed as "ContainerCreating" while the volume is mounted before the status changes to "Running".

3. To verify the PVC is accessible within the Pod, you must gain shell access to your Pod. To start the shell session, execute the following command:
```
kubectl exec -it pvc-demo-pod -- sh
```

4. To create a simple text message as a web page in the Pod enter the following commands:
```
echo Test webpage in a persistent volume!>/var/www/html/index.html
chmod +x /var/www/html/index.html
```

5. Verify the text file contains your message:
```
cat /var/www/html/index.html
```

**Output:**
```
Test webpage in a persistent volume!
```

# Create StatefulSets with PVCs

Use your PVC in a StatefulSet. A StatefulSet is like a Deployment, except that the Pods are given unique identifiers.

### Release the PVC

1. Before you can use the PVC with the statefulset, you must delete the Pod that is currently using it. Execute the following command to delete the Pod:
```
kubectl delete pod pvc-demo-pod
```

2. Confirm the Pod has been removed:
```
kubectl get pods
```

**Output:**
```
No resources found in default namespace.
```

### Create a StatefulSet
Using the manifest file `statefulset-demo.yaml` that creates a StatefulSet that includes a LoadBalancer service and three replicas of a Pod containing an nginx container and a volumeClaimTemplate for 30 gigabyte PVCs with the name `hello-web-disk`. The nginx containers mount the PVC called `hello-web-disk` at `/var/www/html` as in the previous task.

1. To create the StatefulSet with the volume, execute the following command:
```
kubectl apply -f statefulset-demo.yaml
```

**Output:**
```
service "statefulset-demo-service" created
statefulset.apps "statefulset-demo" created
```

You now have a statefulset running behind a service named `statefulset-demo-service`.

### Verify the connection of Pods in StatefulSets
1. Use "kubectl describe" to view the details of the StatefulSet:
```
kubectl describe statefulset statefulset-demo
```
Note the event status at the end of the output. The service and statefulset created successfully.
```
Normal  SuccessfulCreate  10s   statefulset-controller
Message: create Claim hello-web-disk-statefulset-demo-0 Pod statefulset-demo-0 in StatefulSet statefulset-demo success

Normal  SuccessfulCreate  10s   statefulset-controller
Message: create Pod statefulset-demo-0 in StatefulSet statefulset-demo successful
```

2. List the Pods in the cluster:
```
kubectl get pods
```

**Output:**
```
NAME                 READY     STATUS    RESTARTS   AGE
statefulset-demo-0   1/1       Running   0          6m
statefulset-demo-1   1/1       Running   0          3m
statefulset-demo-2   1/1       Running   0          2m
```

3. To list the PVCs, execute the following command:
```
kubectl get pvc
```

**Output:**
```
NAME                          STATUS    VOLUME           CAPACITY ACCESS
hello-web-disk                Bound     pvc-86117ea6-...34   30Gi    RWO
hello-web-disk-st...-demo-0   Bound     pvc-92d21d0f-...34   30Gi    RWO
hello-web-disk-st...-demo-1   Bound     pvc-9bc84d92-...34   30Gi    RWO
hello-web-disk-st...-demo-2   Bound     pvc-a526ecdf-...34   30Gi    RWO
```

The original hello-web-disk is still there and you can now see the individual PVCs that were created for each Pod in the new statefulset Pod.

4. Use "kubectl describe" to view the details of the first PVC in the StatefulSet:
```
kubectl describe pvc hello-web-disk-statefulset-demo-0
```





 
