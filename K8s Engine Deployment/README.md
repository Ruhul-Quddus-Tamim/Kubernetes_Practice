# Objectives

1. Create deployment manifests, deploy to cluster, and verify Pod rescheduling as nodes are disabled.
2. Trigger manual scaling up and down of Pods in deployments.
3. Trigger deployment rollout (rolling update to new version) and rollbacks.
4. Perform a Canary deployment.

# Connect to the GKE cluster
Create a GKE cluster called `autopilot-cluster-1` and open CloudShell in google console

1. In Cloud Shell, type the following command to set the environment variable for the zone and cluster name:

```
export my_region=REGION
export my_cluster=autopilot-cluster-1
```

2. Configure kubectl tab completion in Cloud Shell:

```
source <(kubectl completion bash)
```

3. In Cloud Shell, configure access to your cluster for the kubectl command-line tool, using the following command:

```
gcloud container clusters get-credentials $my_cluster --region $my_region
```

# Create a deployment manifest
You will create a deployment using a sample deployment manifest called nginx-deployment.yaml. This deployment is configured to run three Pod replicas with a single nginx container in each Pod listening on TCP port 80.

To deploy your manifest, execute the following command:

```
kubectl apply -f ./nginx-deployment.yaml
```

To view a list of deployments, execute the following command:

```
kubectl get deployments
```

The output should look like this example.

**Output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           42s
```

# Manually scale up and down the number of Pods in deployments
Sometimes, you want to shut down a Pod instance. Other times, you want ten Pods running. In Kubernetes, you can scale a specific Pod to the desired number of instances. To shut them down, you scale to zero.

### Scale Pods up and down in the console

1. Switch to the Google Cloud Console tab.
2. On the Navigation menu ( Navigation menu icon), click Kubernetes Engine > Workloads.
3. Click nginx-deployment (your deployment) to open the Deployment details page.
4. At the top, click ACTIONS > Scale > Edit Replicas.
5. Type 1 and click SCALE.

This action scales down your cluster. You should see the Pod status being updated under Managed Pods. You might have to click Refresh.

In the Cloud Shell, to view a list of Pods in the deployments, execute the following command:
```
kubectl get deployments
```

**Output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           3m
```

To scale the Pod back up to three replicas, execute the following command:
```
kubectl scale --replicas=3 deployment nginx-deployment
```

To view a list of Pods in the deployments, execute the following command:
```
kubectl get deployments
```

**Output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           4m
```

# Trigger a deployment rollout and a deployment rollback
A deployment's rollout is triggered if and only if the deployment's Pod template (that is, `.spec.template`) is changed, for example, if the labels or container images of the template are updated. Other updates, such as scaling the deployment, do not trigger a rollout.

Here, you trigger deployment rollout, and then you trigger deployment rollback.

### Trigger a deployment rollout
1. To update the version of nginx in the deployment, execute the following command:
```
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.9.1
```
This updates the container image in your Deployment to nginx v1.9.1.

2. To annotate the rollout with details on the change, execute the following command:
```
kubectl annotate deployment nginx-deployment kubernetes.io/change-cause="version change to 1.9.1" --overwrite=true
```

3. To view the rollout status, execute the following command:
```
kubectl rollout status deployment.v1.apps/nginx-deployment
```

The output should look like the example.

**Output:**
```
Waiting for rollout to finish: 1 out of 3 new replicas updated...
Waiting for rollout to finish: 1 out of 3 new replicas updated...
Waiting for rollout to finish: 1 out of 3 new replicas updated...
Waiting for rollout to finish: 2 out of 3 new replicas updated...
Waiting for rollout to finish: 2 out of 3 new replicas updated...
Waiting for rollout to finish: 2 out of 3 new replicas updated...
Waiting for rollout to finish: 1 old replicas pending termination...
Waiting for rollout to finish: 1 old replicas pending termination...
deployment "nginx-deployment" successfully rolled out
```

4. To verify the change, get the list of deployments:
```
kubectl get deployments
```
The output should look like the example.

**Output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           6m
```

5. View the rollout history of the deployment:
```
kubectl rollout history deployment nginx-deployment
```

The output should look like the example. Your output might not be an exact match.

**Output:**
```
deployments "nginx-deployment"
REVISION  CHANGE-CAUSE
1         
2         version change to 1.9.1
```

### Trigger a deployment rollback
To roll back an object's rollout, you can use the kubectl rollout undo command.

1. To roll back to the previous version of the nginx deployment, execute the following command:
```
kubectl rollout undo deployments nginx-deployment
```

2. View the updated rollout history of the deployment:
```
kubectl rollout history deployment nginx-deployment
```

The output should look like the example. Your output might not be an exact match.

**Output:**
```
deployments "nginx-deployment"
REVISION  CHANGE-CAUSE
2        version change to 1.9.1
3         
```

3. View the details of the latest deployment revision:
```
kubectl rollout history deployment/nginx-deployment --revision=3
```

The output should look like the example. Your output might not be an exact match but it will show that the current revision has rolled back to nginx:1.7.9.

**Output:**
```
deployments "nginx-deployment" with revision #3
Pod Template:
  Labels:       app=nginx
        pod-template-hash=3123191453
  Containers:
   nginx:
    Image:      nginx:1.7.9
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        
    Mounts:     
  Volumes:      
```

# Define the service type in the manifest

Here, you create and verify a service that controls inbound traffic to an application. Services can be configured as ClusterIP, NodePort or LoadBalancer types. Now this time, you configure a LoadBalancer.

### Define service types in the manifest
A manifest file called `service-nginx.yaml` that deploys a **LoadBalancer** service type has been provided for you. This service is configured to distribute inbound traffic on TCP port 60000 to port 80 on any containers that have the label app: nginx.

In the Cloud Shell, to deploy your manifest, execute the following command:
```
kubectl apply -f ./service-nginx.yaml
```

### Verify the LoadBalancer creation
To view the details of the nginx service, execute the following command:
```
kubectl get service nginx
```

The output should look like the example.

**Output:**
```
NAME      CLUSTER_IP      EXTERNAL_IP      PORT(S)   SELECTOR    AGE
nginx     10.X.X.X        X.X.X.X          60000/TCP    run=nginx   1m
```

When the external IP appears, open http://[EXTERNAL_IP]:60000/ in a new browser tab to see the server being served through network load balancing.

<img width="817" alt="Screenshot 2024-08-24 at 11 50 15 PM" src="https://github.com/user-attachments/assets/0c1818e2-7cf6-4b9d-b4c0-305ee4131b9b">

# Perform a canary deployment
A canary deployment is a separate deployment used to test a new version of your application. A single service targets both the canary and normal deployments. And it can direct a subset of users to the canary version to mitigate the risk of new releases.

Here, you create a canary deployment to deploys a single pod running a newer version of nginx than your main deployment.

1. Create the canary deployment based on the configuration file:
```
kubectl apply -f ./nginx-canary.yaml
```

2. When the deployment is complete, verify that both the nginx and the nginx-canary deployments are present:
```
kubectl get deployments
```

3. Switch back to the browser tab that is connected to the external LoadBalancer service ip and refresh the page. You should continue to see the standard `Welcome to nginx` page.

4. Switch back to the Cloud Shell and scale down the primary deployment to 0 replicas:
```
kubectl scale --replicas=0 deployment nginx-deployment
```

5. Verify that the only running replica is now the Canary deployment:
```
kubectl get deployments
```

6. Switch back to the browser tab that is connected to the external LoadBalancer service ip and refresh the page. You should continue to see the standard Welcome to nginx page showing that the Service is automatically balancing traffic to the canary deployment.

# Session affinity
The service configuration used in here does not ensure that all requests from a single client will always connect to the same Pod. Each request is treated separately and can connect to either the normal nginx deployment or to the nginx-canary deployment.

This potential to switch between different versions may cause problems if there are significant changes in functionality in the canary release. To prevent this you can set the `sessionAffinity` field to `ClientIP` in the specification of the service if you need a client's first request to determine which Pod will be used for all subsequent connections.

For example - 
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  sessionAffinity: ClientIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 60000
    targetPort: 80
```





