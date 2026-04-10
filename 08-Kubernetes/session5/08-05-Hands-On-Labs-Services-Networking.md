# Session 8-05:  Networking - Services & Service Discovery Labs

This document contains 6 practical labs to build hands-on experience with Kubernetes Services.

---

## Lab 1: Create a ClusterIP Service

### Objective
Deploy a simple nginx Pod and expose it via a ClusterIP Service. Test connectivity from within the cluster.

### Prerequisites
- Kubernetes cluster running (minikube, kind, or EKS)
- kubectl configured
- Basic understanding of Pods and manifests

### Steps

1. **Create an nginx deployment**
   ```bash
   kubectl create deployment nginx-app --image=nginx:latest --replicas=3
   kubectl get pods
   ```
   Expected: 3 nginx pods running

2. **Expose the deployment with a ClusterIP service**

   Create a file named `nginx-clusterip-service.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
   spec:
     type: ClusterIP
     selector:
       app: nginx-app
     ports:
       - port: 80
         targetPort: 80
   ```
   ```bash
   kubectl apply -f nginx-clusterip-service.yaml
   kubectl get svc
   ```
   Expected output shows `nginx-service` with a ClusterIP (e.g., 10.96.x.x)

3. **Inspect the service details**
   ```bash
   kubectl describe svc nginx-service
   ```
   Note:
   - ClusterIP: stable internal IP
   - Endpoints: list of Pod IPs that back this service
   - Port mapping: 80 → 80
   - Selector: app=nginx-app (matches Pod labels)

4. **Test service connectivity from inside the cluster**
   ```bash
   # Get the name of one nginx pod
   POD_NAME=$(kubectl get pods -l app=nginx-app -o jsonpath='{.items[0].metadata.name}')

   # Execute curl from inside the pod to test the service
   kubectl exec -it $POD_NAME -- curl http://nginx-service:80
   ```
   Expected: nginx welcome page HTML output

5. **Test DNS resolution**
   ```bash
   kubectl exec -it $POD_NAME -- nslookup nginx-service
   ```
   Expected: Resolves to the ClusterIP

6. **Test with full FQDN**
   ```bash
   kubectl exec -it $POD_NAME -- curl http://nginx-service.default.svc.cluster.local:80
   ```
   Expected: Same as step 4


### Troubleshooting
- **Service DNS not resolving**: Check CoreDNS pods are running (`kubectl get pods -n kube-system -l k8s-app=kube-dns`)
- **No endpoints shown**: Verify Pod labels match service selector (`kubectl get pods --show-labels`)
- **Connection refused**: Ensure target-port matches Pod container port (80 for nginx)

---

## Lab 2: Create a NodePort Service

### Objective
Expose the nginx application on a NodePort so it's accessible from outside the cluster.

### Prerequisites
- Lab 1 completed (nginx deployment running)
- Access to a worker node IP
- Network access to worker nodes

### Steps

1. **Create a NodePort service**

   Create a file named `nginx-nodeport-service.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-nodeport
   spec:
     type: NodePort
     selector:
       app: nginx-app
     ports:
       - port: 80
         targetPort: 80
         # nodePort is optional - omit to let Kubernetes auto-assign in range 30000-32767
   ```
   ```bash
   kubectl apply -f nginx-nodeport-service.yaml
   kubectl get svc
   ```
   Expected output shows `nginx-nodeport` with NodePort (30000-32767)

2. **Get the assigned NodePort**
   ```bash
   NODE_PORT=$(kubectl get svc nginx-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
   echo "NodePort: $NODE_PORT"
   ```
   Example: 30845

3. **Get a worker node IP**
   ```bash
   NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
   echo "Node IP: $NODE_IP"
   ```

4. **Access the service from outside the cluster**

   Pre-req: Make sure that the node's Security Group in AWS has NodePort added in the Inbound rule from your IP or from everywhere for demo purpose

   ```bash
   # Replace with your actual IP and port
   curl http://<NODE_IP>:<NODE_PORT>
   ```
   or Open it in the browser

   Expected: nginx welcome page HTML output

5. **Verify the service has both ClusterIP and NodePort**
   ```bash
   kubectl describe svc nginx-nodeport
   ```
   Note:
   - ClusterIP: Still has internal IP
   - Port: 80/TCP (internal)
   - NodePort: 30xxx/TCP (external)

6. **Test round-robin across nodes**
   ```bash
   # If using minikube, test local access
   minikube ssh
   curl localhost:<NODE_PORT>
   # or from host
   curl http://$(minikube ip):<NODE_PORT>
   ```

---

## Lab 3: Create a LoadBalancer Service

### Objective
Expose nginx via a cloud provider LoadBalancer (for EKS)

### Prerequisites
- Running on cloud Kubernetes (EKS)

### Steps

1. **Create a LoadBalancer service**

   Create a file named `nginx-lb-service.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-lb
   spec:
     type: LoadBalancer
     selector:
       app: nginx-app
     ports:
       - port: 80
         targetPort: 80
   ```
   ```bash
   kubectl apply -f nginx-lb-service.yaml
   kubectl get svc
   ```
   Expected: Service shows `<pending>` for EXTERNAL-IP initially

2. **Wait for external IP to be assigned**
   ```bash
   # Watch for EXTERNAL-IP to appear
   kubectl get svc nginx-lb --watch

   # For EKS: Usually assigned within 1-2 minutes
   ```

3. **Get the external IP and access the service**
   ```bash
   EXTERNAL_IP=$(kubectl get svc nginx-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo "External IP: $EXTERNAL_IP"

   curl http://$EXTERNAL_IP:80
   ```
   Expected: nginx welcome page HTML output

4. **Inspect the service details**
   ```bash
   kubectl describe svc nginx-lb
   ```
   Note:
   - ClusterIP: Still present for internal routing
   - Port: 80/TCP
   - NodePort: Also assigned (30000-32767)
   - LoadBalancer Ingress: Shows cloud LB IP or hostname (EKS provisions an NLB by default)

5. **Verify routing path**
   - External request hits cloud Layer 4 load balancer (e.g., AWS NLB)
   - LB forwards to node on NodePort
   - kube-proxy forwards to Pod on target-port

---

## Lab 4: Service Discovery with DNS

### Objective
Understand how Kubernetes DNS resolves service names and test various FQDN formats.

### Prerequisites
- Lab 1-3 completed (services running)
- kubectl configured

### Steps

1. **Verify CoreDNS is running**
   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   # or for newer versions
   kubectl get deployment -n kube-system coredns
   ```
   Expected: CoreDNS pod(s) in Running state

2. **Deploy a test pod for DNS queries**
   ```bash
   kubectl run -it dns-test --image=alpine --restart=Never -- sh

   # Inside the pod:
   apk add --no-cache dnsutils curl
   ```

3. **Test DNS resolution with short name**
   ```bash
   # Still inside dns-test pod
   nslookup nginx-service
   # or using dig
   dig nginx-service
   ```
   Expected: Resolves to the ClusterIP (10.96.x.x)

4. **Test DNS with namespace**
   ```bash
   nslookup nginx-service.default
   ```
   Expected: Same IP as short name

5. **Test full FQDN**
   ```bash
   nslookup nginx-service.default.svc.cluster.local
   ```
   Expected: Same IP as previous tests

6. **Test service connectivity via DNS name**
   ```bash
   curl http://nginx-service
   curl http://nginx-service.default.svc.cluster.local:80
   ```
   Expected: nginx welcome page

7. **Test DNS for service in different namespace**
   ```bash
   # Create another namespace
   kubectl create namespace dev

   # Create a ClusterIP service in the dev namespace
   ```
   Create a file named `web-service-dev.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: web-service
     namespace: dev
   spec:
     type: ClusterIP
     selector:
       app: nginx-app
     ports:
       - port: 80
         targetPort: 80
   ```
   ```bash
   kubectl apply -f web-service-dev.yaml

   # From dns-test pod, resolve it
   nslookup web-service.dev.svc.cluster.local
   ```
   Expected: Different ClusterIP from the default namespace service

8. **Test DNS timeout for non-existent service**
   ```bash
   nslookup nonexistent-service
   ```
   Expected: NXDOMAIN or timeout

9. **Exit the test pod**
   ```bash
   exit
   kubectl delete pod dns-test
   ```

### Troubleshooting
- **DNS resolution fails**: Check CoreDNS logs: `kubectl logs -n kube-system -l k8s-app=kube-dns`
- **Cannot resolve other namespace service**: Must use full FQDN: `service.namespace.svc.cluster.local`
---