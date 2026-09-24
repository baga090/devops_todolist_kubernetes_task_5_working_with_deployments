# ToDo App Kubernetes Deployment & Autoscaling

## 1. How to deploy the app to k8s
To deploy the application, apply all manifests in the `.infrastructure` directory (this will create the `mateapp` namespace, the Deployment, and the HPA):
```bash
kubectl apply -f .infrastructure/
```
*Verification:* Wait a few seconds and verify that the pods are running and the HPA is active:
```bash
kubectl get pods -n mateapp
kubectl get hpa -n mateapp
```

## 2. Resource Requests and Limits Explanation
* **Requests (`cpu: 100m`, `memory: 128Mi`):** This is the guaranteed baseline required for the lightweight Django app to start and idle comfortably. It helps the Kubernetes scheduler find appropriate nodes with available capacity.
* **Limits (`cpu: 250m`, `memory: 256Mi`):** This sets a hard cap to prevent a single pod from monopolizing node resources during spikes. Because requests are lower than limits, the pods get a **Burstable QoS** class, providing a good balance between performance and resource efficiency until HPA scales out.

## 3. HPA Configuration Explanation
The Horizontal Pod Autoscaler is configured with:
* **minReplicas: 2:** Satisfies the requirement for high availability and maintaining 2 running pods in an idle state.
* **maxReplicas: 5:** Sets a reasonable ceiling to control cluster resource consumption during massive traffic bursts.
* **Triggers (CPU at 70%, Memory at 80%):** Scaling is triggered proactively *before* hitting the absolute container limits (250m/256Mi). This ensures new pods are ready to take the load before existing ones start throttling (CPU) or get terminated by the OOMKiller (Memory).

## 4. Strategy Configuration (RollingUpdate)
We use the `RollingUpdate` strategy with:
* **maxSurge: 1**
* **maxUnavailable: 1**
* **Why these numbers:** Since our minimum replica count is 2, setting `maxUnavailable: 1` guarantees that at least 1 pod (50% capacity) is always running and serving traffic during an update, ensuring zero downtime. `maxSurge: 1` ensures we don't overwhelm the cluster nodes by spinning up too many new pods simultaneously during the rollout.

## 5. How to access the app after deployment
Since this specific task focuses on Deployment and HPA without explicitly defining a ClusterIP/NodePort Service, you can access the application by directly port-forwarding the Deployment to your local machine:
```bash
kubectl port-forward -n mateapp deployment/todoapp-deployment 8080:8000
```
After running this command, open your web browser and navigate to:
`http://localhost:8080`
