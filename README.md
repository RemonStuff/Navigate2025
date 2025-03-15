# Serverless Computing with Kubernetes Demo

## Introduction 

## Setup 

### 1. Prerequisites verification

```powershell
minikube version
```
*This confirms Minikube is installed and shows its version.*

```powershell
kubectl version --client
```
*This verifies kubectl is installed and ready to communicate with our cluster.*

```powershell
Get-WindowsOptionalFeature -FeatureName Microsoft-Hyper-V-All -Online
```
*This checks if Hyper-V is enabled on your Windows system. We need Hyper-V as it provides better performance than alternatives for running Minikube.*

### 2. Start Minikube cluster with Hyper-V

```powershell
minikube start --driver=hyperv --memory=6144  --cpus=4
```
*This launches a Minikube cluster with 6GB RAM and 4 CPU cores, providing sufficient resources for our demo without overwhelming your laptop.*
We'll demonstrate:

1. How serverless workloads can run on any Kubernetes cluster, including local development environments
2. The operational benefits of serverless: scale-to-zero, automatic scaling, and resource efficiency
3. The cost advantages compared to traditional deployments with fixed capacity
4. How this approach provides cloud-native flexibility without vendor lock-in

#### What is serverless computing?
Serverless computing is a cloud execution model where developers build and run applications without managing servers - the infrastructure is automatically provisioned, scaled, and maintained by the platform. Despite the name, servers still exist, but they're abstracted away; developers simply deploy code while the system handles scaling (including scaling to zero when idle), infrastructure management, and resource allocation. This model offers benefits like reduced operational overhead, automatic scaling based on demand, and pay-for-what-you-use pricing, making it ideal for variable workloads and allowing development teams to focus on code rather than infrastructure.

### 3. Install Knative Serving
Knative Serving is an open-source Kubernetes extension that implements serverless capabilities on any Kubernetes cluster. It's the industry standard for implementing serverless workloads on Kubernetes, providing key features like scale-to-zero (eliminating costs when idle), request-based autoscaling, revision management, and traffic splitting. Unlike proprietary serverless options such as AWS Lambda or Azure Functions, Knative works on any Kubernetes cluster whether on-premises, in public clouds, or on local development environments, which eliminates vendor lock-in while still delivering the operational benefits of serverless computing.

```powershell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-crds.yaml
```
*This installs the Custom Resource Definitions (CRDs) for Knative Serving, which define the serverless objects we'll use.*

```powershell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-core.yaml
```
*This installs the core Knative Serving components that will manage our serverless workloads.*

### 4. Install Kourier as networking layer

Kourier is a lightweight, high-performance ingress, that we will use for Knative Serving. It will handles external traffic routing to serverless applications. We're using it instead of alternatives like Istio because it's significantly lighter weight (less than 20MB versus Istio's 300MB+), starts up much faster (seconds versus minutes), and focuses solely on networking without the complexity of a full service mesh, making it ideal for demonstrations and development environments where quick setup and minimal resource usage are priorities.

```powershell
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.13.0/kourier.yaml
```

### 5. Configure Knative to use Kourier

```powershell
kubectl get configmap config-network -n knative-serving -o yaml > config-network.yaml
```
*This exports the current Knative network configuration to a file so we can edit it.*

```powershell
code config-network.yaml
```
*This opens the configuration file in VS Code. Edit the file to include the following in the data section:*

```yaml
data:
  ingress-class: "kourier.ingress.networking.knative.dev"
  # (keep any other existing data entries)
```

```powershell
kubectl apply -f config-network.yaml
```
*This applies our updated configuration, telling Knative to use Kourier for handling incoming traffic.*

### 6. Configure DNS

```powershell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-default-domain.yaml
```
*This sets up Knative's default DNS configuration, making our serverless applications accessible via predictable URLs during the demo.*

## Demo Part 1: Basic Serverless Application (10 minutes)

### 1. Create a simple serverless application

Create a new file in VS Code named `hello-service.yaml` with this content:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-serverless
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          env:
            - name: TARGET
              value: "Kubernetes Serverless Demo"
```
This defines a serverless service using Knative's custom resource type Service. Let's break down the key components:
- apiVersion: serving.knative.dev/v1: Specifies the Knative API being used
- kind: Service: Defines this as a Knative Service, which automatically manages the entire lifecycle of a workload
- metadata.name: Names our service "hello-serverless", which affects the generated URL
- spec.template.spec.containers: Defines the container to run, similar to Kubernetes Pod definitions
- image: gcr.io/knative-samples/helloworld-go: A sample Go application that responds with a customizable message
- env.TARGET: Environment variable that the sample app uses to customize its response
Unlike traditional Kubernetes deployments, this single resource automatically creates and manages multiple Kubernetes objects behind the scenes: Deployments, ReplicaSets, Pods, Services, and network configurations. It also handles scale-to-zero, autoscaling, revisions, and routing without requiring you to configure these aspects separately.

### 2. Deploy the application

```powershell
kubectl apply -f hello-service.yaml
```
This warning is about container security contexts. By default, Kubernetes containers have certain security-related permissions that Knative considers too permissive. The warning indicates that:
1. We haven't specified security constraints in our YAML
2. Kubernetes is using its default (more permissive) security settings
3. Knative is warning that in future releases, it may automatically apply stricter security defaults
For a demo environment this warning is acceptable, but in production, you should explicitly define security contexts following the principle of least privilege. You can add security context settings to your YAML to eliminate this warning.

### 3. Demonstrate zero-to-running

```powershell
kubectl get pods -w
```
*The -w flag gives us a real-time view of pods being created. This shows how quickly our serverless application starts from zero - demonstrating the "cold start" process that happens when a request first arrives.*

### 4. Access the serverless application

```powershell
$URL = kubectl get ksvc hello-serverless -o jsonpath='{.status.url}'
```
*This retrieves the URL that Knative assigned to our service.*

```powershell
Write-Host "Service URL: $URL"
```
*This displays the service URL for reference.*

```powershell
# Before we can access the service, we need to set up port forwarding
Write-Host "Setting up port forwarding for Kourier gateway..."
```

```powershell
# Start port forwarding in a separate PowerShell window (you can run this in a new terminal)
Start-Process powershell -ArgumentList "-Command kubectl port-forward -n kourier-system svc/kourier 8080:80"
```
*This creates a port forward from your local machine's port 8080 to the Kourier service in the cluster.*

```powershell
Write-Host "Waiting for port forward to establish..."
Start-Sleep -Seconds 3
```
*Gives the port forward a moment to establish.*

```powershell
# Extract just the hostname part from the Knative URL
$HOST_HEADER = ($URL -replace "http://", "") -replace "/.*", ""
```
*Extracts the host header we need to use for routing through Knative.*

```powershell
Write-Host "Testing connection to serverless application..."
```

```powershell
# Access the service through the port forward with the proper host header
Invoke-WebRequest -Uri "http://localhost:8080" -Headers @{Host=$HOST_HEADER} | Select-Object -ExpandProperty Content
```
*This sends a request to our serverless application via the port forward, including the required Host header, and displays the response.*

```powershell
Write-Host "`nAlternatively, for easier demonstration, you can also use:"
```

```powershell
# Alternative for demo purposes: Access via Minikube IP and NodePort
$MINIKUBE_IP = minikube ip
$KOURIER_PORT = kubectl get svc kourier -n kourier-system -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
```
*This gets the Minikube IP address and the NodePort that Kourier is using to expose our services.*

```powershell
Write-Host "Direct NodePort access: http://$MINIKUBE_IP`:$KOURIER_PORT"
```

```powershell
Invoke-WebRequest -Uri "http://$MINIKUBE_IP`:$KOURIER_PORT" -Headers @{Host=$HOST_HEADER} | Select-Object -ExpandProperty Content
```
*This provides a second way to access the application, directly through the NodePort, which can be more stable in some demo environments.*

```powershell
Write-Host "`nNote: In both cases, the Host header is crucial for Knative routing to work properly."
```
*This emphasizes the importance of the Host header when working with Knative services.*

## Demo Part 2: Autoscaling Benefits 

### 1. Configure autoscaling for the service

Create a new file in VS Code named `hello-service-scaled.yaml` with this content:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello-serverless
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "5"
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          env:
            - name: TARGET
              value: "Kubernetes Serverless Demo"
```

*We're adding autoscaling parameters using Knative annotations: minScale=1 prevents cold starts, maxScale=5 limits resource usage, and target=10 means each instance handles about 10 concurrent requests before scaling out.*

```powershell
kubectl apply -f hello-service-scaled.yaml
```
*This updates our serverless service with the new autoscaling configuration.*

### 2. Install load testing tool on Windows

```powershell
winget install hey
```
*This installs the "hey" load testing tool via Windows package manager.*

*Alternatively, if winget isn't available:*

```powershell
Invoke-WebRequest -Uri "https://hey-release.s3.us-east-2.amazonaws.com/hey_windows_amd64" -OutFile "hey.exe"
```
*This downloads the hey.exe tool directly, which we'll use to generate load for our autoscaling demo.*

### 3. Load test the application

```powershell
$MINIKUBE_IP = minikube ip
```
*Gets the Minikube IP address again.*

```powershell
$KOURIER_PORT = kubectl get svc kourier -n kourier-system -o jsonpath="{.spec.ports[?(@.name=='http2')].nodePort}"
```
*Gets the Kourier port again.*

```powershell
.\hey.exe -z 30s -c 50 http://$MINIKUBE_IP`:$KOURIER_PORT/hello-serverless
```
*This simulates 50 concurrent users hitting our application for 30 seconds, which will trigger Knative's autoscaling mechanism.*

### 4. Observe autoscaling in action

```powershell
kubectl get pods -w
```
*Open this in a new PowerShell window before running the load test. As traffic increases, you'll see Knative automatically create additional pods to handle the load, demonstrating how serverless adapts to demand changes without manual intervention.*

```powershell
kubectl delete -f hello-service-scaled.yaml
```

## Demo Part 3: Resource Efficiency (10 minutes)

### 1. Deploy traditional deployment for comparison

Create a new file in VS Code named `traditional-deployment.yaml` with this content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-traditional
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-traditional
  template:
    metadata:
      labels:
        app: hello-traditional
    spec:
      containers:
      - name: hello
        image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Traditional Deployment"
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: hello-traditional
spec:
  selector:
    app: hello-traditional
  ports:
  - port: 80
    targetPort: 8080
  type: NodePort
```

*This creates a traditional Kubernetes deployment with 3 fixed replicas for comparison. Notice how much more configuration is required compared to our serverless definition.*

```powershell
kubectl apply -f traditional-deployment.yaml
```
*This deploys our traditional Kubernetes application alongside our serverless one.*

### 2. Compare resource usage

```powershell
minikube addons enable metrics-server
```
*This enables the metrics-server addon in Minikube, which we need to measure resource usage.*

```powershell
Start-Sleep -Seconds 30
```
*This waits 30 seconds for the metrics to become available.*

```powershell
kubectl top pods
```
*This shows the actual CPU and memory usage of all pods. The traditional deployment will show consistent usage across all pods, while the serverless deployment will show resources scaling with actual traffic.*

### 3. Show dashboard and metrics

```powershell
minikube dashboard
```
*This opens the Kubernetes dashboard in your browser, providing a visual representation of your cluster that makes it easier to compare the two deployment types.*

### 4. Resource efficiency calculation

```powershell
$TRAD_CPU = 3 * 100
$TRAD_MEM = 3 * 64
$SERVERLESS_PODS = (kubectl get pods --selector=serving.knative.dev/service=hello-serverless --no-headers | Measure-Object -Line).Lines
$SERVERLESS_CPU = $SERVERLESS_PODS * 100
$SERVERLESS_MEM = $SERVERLESS_PODS * 64
$CPU_SAVING = ($TRAD_CPU - $SERVERLESS_CPU) / $TRAD_CPU * 100
$MEM_SAVING = ($TRAD_MEM - $SERVERLESS_MEM) / $TRAD_MEM * 100

Write-Host "Resource Efficiency Analysis:"
Write-Host "Traditional deployment: $TRAD_CPU mCPU, $TRAD_MEM Mi Memory (constantly allocated)"
Write-Host "Serverless deployment: $SERVERLESS_CPU mCPU, $SERVERLESS_MEM Mi Memory (scales with usage)"
Write-Host "Potential savings when idle: $CPU_SAVING% CPU, $MEM_SAVING% Memory"
```
*Shows percentage savings when traffic is low or zero, highlighting serverless cost benefits.*

## Cleanup (2 minutes)

### 1. Remove deployed applications

```powershell
kubectl delete -f hello-service-scaled.yaml
```
*This removes our serverless application and all associated resources created by Knative.*

```powershell
kubectl delete -f traditional-deployment.yaml
```
*This removes our traditional deployment and its associated service.*

### 2. Uninstall Knative components

```powershell
kubectl delete -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-default-domain.yaml
```
*This removes the Knative default domain configuration.*

```powershell
kubectl delete -f https://github.com/knative/net-kourier/releases/download/knative-v1.13.0/kourier.yaml
```
*This removes the Kourier networking layer.*

```powershell
kubectl delete -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-core.yaml
```
*This removes the Knative Serving core components.*

```powershell
kubectl delete -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-crds.yaml
```
*This removes the Knative Custom Resource Definitions.*

### 3. Stop Minikube

```powershell
minikube stop
```
*This stops the Minikube virtual machine, preserving its state for future demos.*

```powershell
minikube delete
```
*Optional: This completely removes the Minikube VM and all associated data if you don't need it anymore.*

## Conclusion

This demo has shown how serverless computing on Kubernetes combines the best of both worlds:

1. The operational simplicity of serverless (automatic scaling, no capacity planning)
2. The flexibility of Kubernetes (runs anywhere, no vendor lock-in)
3. Cost efficiency through true scale-to-zero capabilities
4. Simplified developer experience with less infrastructure code

For production deployments, the same principles apply - consider using tools like Knative, OpenFaaS, or Kubeless to bring serverless advantages to your Kubernetes environments.
