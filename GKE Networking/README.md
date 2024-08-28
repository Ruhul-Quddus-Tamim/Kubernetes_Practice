# Create a cluster network policy
Create a cluster network policy to restrict communication between the Pods. A zero-trust zone is important to prevent lateral attacks within the cluster when an intruder compromises one of the Pods.

### Create a GKE cluster
1. In Cloud Shell, type the following command to set the environment variable for the zone and cluster name:
```
export my_zone=ZONE
export my_cluster=standard-cluster-1
```

2. Configure kubectl tab completion in Cloud Shell:
```
source <(kubectl completion bash)
```

3. In Cloud Shell, type the following command to create a Kubernetes cluster. Note that this command adds the additional flag --enable-network-policy to the parameters you have used in previous labs. This flag allows this cluster to use cluster network policies:
```
gcloud container clusters create $my_cluster --num-nodes 3 --enable-ip-alias --zone $my_zone --enable-network-policy
```

4. In Cloud Shell, configure access to your cluster for the kubectl command-line tool, using the following command:
```
gcloud container clusters get-credentials $my_cluster --zone $my_zone
```

5. Run a simple web server application with the label app=hello, and expose the web application internally in the cluster:
```
kubectl run hello-web --labels app=hello \
  --image=gcr.io/google-samples/hello-app:1.0 --port 8080 --expose
```

### Restrict incoming traffic to Pods
This manifest file defines an ingress policy that allows access to Pods labeled `app: hello` from Pods labeled `app: foo`.

1. Create an ingress policy:
```
kubectl apply -f hello-allow-from-foo.yaml
```

2. Verify that the policy was created:
```
kubectl get networkpolicy
```

**Output:**
```
NAME                   POD-SELECTOR   AGE
hello-allow-from-foo   app=hello      7s
```

### Validate the ingress policy

1. Run a temporary Pod called test-1 with the label app=foo and get a shell in the Pod:
```
kubectl run test-1 --labels app=foo --image=alpine --restart=Never --rm --stdin --tty
```

> Note: The kubectl switches used here in conjunction with the run command are important to note.
> 
> --stdin ( alternatively -i ) creates an interactive session attached to STDIN on the container.
> 
> --tty ( alternatively -t ) allocates a TTY for each container in the pod.
> 
> --rm instructs Kubernetes to treat this as a temporary Pod that will be removed as soon as it completes its startup task. As this is an interactive session it will be removed as soon as the user exits the session.
> 
> --label ( alternatively -l ) adds a set of labels to the pod.
> 
> --restart defines the restart policy for the Pod.

2. Make a request to the hello-web:8080 endpoint to verify that the incoming traffic is allowed:
```
wget -qO- --timeout=2 http://hello-web:8080
```

**Output:**
```
If you don't see a command prompt, try pressing enter.
/ # wget -qO- --timeout=2 http://hello-web:8080
Hello, world!
Version: 1.0.0
Hostname: hello-web-8b44b849-k96lh
/ #
```

3. Now run a different Pod using the same Pod name but using a label, `app=other`, that does not match the podSelector in the active network policy. This Pod should not have the ability to access the hello-web application.
```
kubectl run test-1 --labels app=other --image=alpine --restart=Never --rm --stdin --tty
```

4. Make a request to the hello-web:8080 endpoint to verify that the incoming traffic is not allowed:
```
wget -qO- --timeout=2 http://hello-web:8080
```
The request times out.

**Output:**
```
If you don't see a command prompt, try pressing enter.
/ # wget -qO- --timeout=2 http://hello-web:8080
wget: download timed out
/ #
```

# Restrict outgoing traffic from the Pods
You can restrict outgoing (egress) traffic as you do incoming traffic. However, in order to query internal hostnames (such as hello-web) or external hostnames (such as www.example.com), you must allow DNS resolution in your egress network policies. DNS traffic occurs on port 53, using TCP and UDP protocols.

This file - `foo-allow-to-hello.yaml` defines a policy that permits Pods with the label `app: foo` to communicate with Pods labeled `app: hello` on any port number, and allows the Pods labeled app: foo to communicate to any computer on UDP port 53, which is used for DNS resolution. Without the DNS port open, you will not be able to resolve the hostnames.

1. Create an egress policy:
```
kubectl apply -f foo-allow-to-hello.yaml
```

2. Verify that the policy was created:
```
kubectl get networkpolicy
```

**Output:**
```
NAME                   POD-SELECTOR   AGE
foo-allow-to-hello     app=foo        7s
hello-allow-from-foo   app=hello      5m
```

### Validate the egress policy
1. Deploy a new web application called hello-web-2 and expose it internally in the cluster:
```
kubectl run hello-web-2 --labels app=hello-2 \
  --image=gcr.io/google-samples/hello-app:1.0 --port 8080 --expose
```

2. Run a temporary Pod with the app=foo label and get a shell prompt inside the container:
```
kubectl run test-3 --labels app=foo --image=alpine --restart=Never --rm --stdin --tty
```

3. Verify that the Pod can establish connections to hello-web:8080:
```
wget -qO- --timeout=2 http://hello-web:8080
```

**Output:**
```
If you don't see a command prompt, try pressing enter.
/ # wget -qO- --timeout=2 http://hello-web:8080
Hello, world!
Version: 1.0.0
Hostname: hello-web
/ #
```

4. Verify that the Pod cannot establish connections to `hello-web-2:8080`:
```
wget -qO- --timeout=2 http://hello-web-2:8080
```

This fails because none of the Network policies you have defined allow traffic to Pods labeled `app: hello-2`.

5. Verify that the Pod cannot establish connections to external websites, such as `www.example.com`:
```
wget -qO- --timeout=2 http://www.example.com
```

This fails because the network policies do not allow external http traffic (tcp port 80).





