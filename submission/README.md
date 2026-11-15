# Microservices Kubernetes Deployment Assessment

**Assessment:** AWS Skill Test 2  
**Repository:** [chinmayvyas7/Microservices-Task](https://github.com/chinmayvyas7/Microservices-Task)  
**Platform:** Windows 11, Docker Desktop, Minikube and Kubernetes  
**Tested on:** 15 November 2026

## 1. Objective

Deploy the four supplied Node.js microservices on a local Kubernetes cluster using Minikube. Configure Deployments, ClusterIP Services, health probes, resource requests and limits, and internal service discovery. Validate the running application through Kubernetes commands, logs and gateway API requests.

## 2. Application architecture

| Component | Container image | Application / Service port | Kubernetes Service type |
|---|---|---:|---|
| User Service | `user-service:assessment` | 3000 | ClusterIP |
| Product Service | `product-service:assessment` | 3001 | ClusterIP |
| Order Service | `order-service:assessment` | 3002 | ClusterIP |
| Gateway Service | `gateway-service:assessment` | 3003 | ClusterIP |

The Gateway handles `/api/users`, `/api/products` and `/api/orders`. It reaches the backend Services using Kubernetes DNS names (`user-service`, `product-service` and `order-service`) and their respective ports. For host testing, the Gateway's Kubernetes Service is port-forwarded to `localhost:8080`.

```text
Host browser / curl
       |
 localhost:8080
       | kubectl port-forward 8080:3003
       v
 gateway-service (ClusterIP:3003)
       |
       +--> user-service    (ClusterIP:3000)
       +--> product-service (ClusterIP:3001)
       +--> order-service   (ClusterIP:3002)
```

The Kubernetes Deployments have one replica each, matching `app` labels/selectors, readiness and liveness HTTP probes at `/health`, and the following resource settings per container:

| Resource | Request | Limit |
|---|---:|---:|
| CPU | `50m` | `250m` |
| Memory | `64Mi` | `256Mi` |

The manifests specify `NODE_ENV=production` and the corresponding `PORT` for each container. The Gateway also sets `NO_PROXY` for Kubernetes-internal service addresses. Images use `imagePullPolicy: Never` because they are built locally and loaded explicitly into Minikube; a public container registry is not needed.

## 3. Prerequisites and tested environment

- Windows 11 with Command Prompt or PowerShell
- Docker Desktop running in **Linux containers** mode
- Minikube **v1.39.0**, Docker driver
- Kubernetes **v1.37.0** (Minikube cluster)
- `kubectl` installed and configured for the `minikube` context
- Git and an internet connection for initial dependencies / base images

Tested Minikube allocation: **4 CPUs** and **4096 MB memory**.

> The application source and its four Dockerfiles are already included in this fork under `Microservices/`. No helper script and no additional Dockerfile creation are necessary.

## 4. Repository structure

```text
Microservices-Task/
├── Microservices/
│   ├── user-service/
│   ├── product-service/
│   ├── order-service/
│   ├── gateway-service/
│   └── docker-compose.yml
├── screenshots/                     # Original repository evidence (Docker Compose)
├── submission/
│   ├── deployments/
│   │   ├── user-service.yaml
│   │   ├── product-service.yaml
│   │   ├── order-service.yaml
│   │   └── gateway-service.yaml
│   ├── services/
│   │   ├── user-service.yaml
│   │   ├── product-service.yaml
│   │   ├── order-service.yaml
│   │   └── gateway-service.yaml
│   ├── screenshots/                 # Actual Minikube assessment evidence
│   └── README.md
└── README.md
```

## 5. Start and verify Minikube

Run the following commands in **CMD or PowerShell**:

```powershell
minikube start --driver=docker --cpus=4 --memory=4096
minikube status
kubectl get nodes
```

Observed result: the `minikube` control-plane node was **Ready**.

## 6. Build and load Docker images

From the **repository root**, build each image using the Dockerfile already provided with its Node.js service:

```powershell
docker build -t user-service:assessment ./Microservices/user-service
docker build -t product-service:assessment ./Microservices/product-service
docker build -t order-service:assessment ./Microservices/order-service
docker build -t gateway-service:assessment ./Microservices/gateway-service
```

Confirm the local images:

```powershell
docker images --filter "reference=*:assessment"
```

Load all four images into the Minikube container runtime:

```powershell
minikube image load user-service:assessment
minikube image load product-service:assessment
minikube image load order-service:assessment
minikube image load gateway-service:assessment
minikube image ls
```

Observed: all four image names were listed by `minikube image ls`.

## 7. Apply Kubernetes manifests

From the **repository root**:

```powershell
kubectl apply --dry-run=client -f submission/services/
kubectl apply --dry-run=client -f submission/deployments/

kubectl apply -f submission/services/
kubectl apply -f submission/deployments/
```

Check deployment readiness:

```powershell
kubectl rollout status deployment/user-service --timeout=120s
kubectl rollout status deployment/product-service --timeout=120s
kubectl rollout status deployment/order-service --timeout=120s
kubectl rollout status deployment/gateway-service --timeout=120s

kubectl get deployments
kubectl get pods -o wide
kubectl get svc
```

### Verified Kubernetes status

| Deployment | Ready | Available | Pod status | Restarts |
|---|---:|---:|---|---:|
| `user-service` | 1/1 | 1 | Running | 0 |
| `product-service` | 1/1 | 1 | Running | 0 |
| `order-service` | 1/1 | 1 | Running | 0 |
| `gateway-service` | 1/1 | 1 | Running | 0 |

All four application Services were created as **ClusterIP** with ports **3000**, **3001**, **3002** and **3003**, respectively. The `kubernetes` ClusterIP Service on port 443 is a system service, not an additional application microservice.

## 8. Test the gateway and service communication

In one terminal, start port-forwarding and **leave it running**:

```powershell
kubectl port-forward service/gateway-service 8080:3003
```

In a **second** CMD or PowerShell terminal:

```powershell
curl.exe http://localhost:8080/health
curl.exe http://localhost:8080/api/users
curl.exe http://localhost:8080/api/products
curl.exe http://localhost:8080/api/orders
```

### Verified HTTP responses

**Gateway health:**

```json
{"status":"Gateway Service is healthy"}
```

**Users:**

```json
[{"id":1,"name":"John Doe"},{"id":2,"name":"Jane Smith"}]
```

**Products:**

```json
[{"id":1,"name":"Laptop","price":999},{"id":2,"name":"Phone","price":699}]
```

**Orders:**

```json
[]
```

All four requests completed successfully. The empty orders array is a valid response for an orders collection with no records. Successful backend responses through the Gateway support that inter-service routing via the configured Kubernetes Services works. Order creation was not part of the Kubernetes test shown in the screenshots.

## 9. Inspect application logs

```powershell
kubectl logs deployment/gateway-service --tail=50
kubectl logs deployment/user-service --tail=20
kubectl logs deployment/product-service --tail=20
kubectl logs deployment/order-service --tail=20
```

Observed startup messages:

```text
Gateway service running on port 3003
User service running on port 3000
Product service running on port 3001
Order service running on port 3002
```

These logs verify startup; the gateway requests in Section 8 provide the HTTP communication evidence. The application does not necessarily log every request.

## 10. Screenshot evidence

The following screenshots were captured in the local assessment environment and are stored under `submission/screenshots/`:

| File | Evidence |
|---|---|
| `01-minikube-node.png` | Minikube control-plane node in `Ready` state |
| `02-docker-images.png` | Four locally built `:assessment` Docker images |
| `03-kubernetes-deployment-overview.png` | Manifest application, four ready Deployments, four running Pods, and ClusterIP Services |
| `04-gateway-service-tests.png` | Successful `/health`, `/api/users`, `/api/products`, and `/api/orders` responses through port-forwarding |
| `05-kubernetes-service-logs.png` | Startup messages from all four microservices |

### Test summary

The Minikube node was `Ready`. All four Deployments were available, and each Pod was `Running` with `1/1` containers ready and no restarts. The application Services were created with the expected ClusterIP ports.

After port-forwarding the Gateway, `/health` returned a healthy response, `/api/users` returned two users, `/api/products` returned two products and `/api/orders` returned an empty list. The four log commands showed each service starting on its assigned port. The API responses, rather than the startup logs alone, are the evidence for gateway-to-backend communication.

The pre-existing root-level `screenshots/` folder contains original Docker Compose evidence and is **not** being represented as Kubernetes deployment evidence.

## 11. Troubleshooting

| Symptom | Suggested checks / fix |
|---|---|
| `kubectl` not recognized in Windows CMD | Open a terminal where it is on `PATH`; alternatively use `minikube kubectl -- get pods` |
| Minikube node is not Ready | `minikube status`; verify Docker Desktop engine is running |
| `ErrImageNeverPull` | Check `minikube image ls` and the exact `:assessment` name; repeat `minikube image load <service>:assessment` |
| Pod stuck in `Pending` | Use `kubectl describe pod <pod-name>`; inspect CPU and memory capacity |
| `CrashLoopBackOff` | Use `kubectl logs deployment/<service>` and `kubectl describe pod <pod-name>`; confirm image and app startup |
| Pod Running but not Ready | Check `/health` probe settings and `kubectl describe pod <pod-name>` |
| Gateway API fails | Check backend Pods/Services, inspect `kubectl get endpointslices`, and confirm service names/ports |
| Port-forward does not start | Confirm Gateway Pod is Ready; use another free local port if 8080 is occupied |
| Image pulls / registry connection fail | The application images are local: ensure they were loaded into Minikube; internet may still be necessary for Minikube/base image setup |

## 12. Submission

From the **repository root**, create the requested ZIP using PowerShell:

```powershell
Compress-Archive -Path .\submission -DestinationPath .\submission.zip -Force
```

The ZIP should contain the `submission/` directory, its four Deployment YAMLs, four Service YAMLs, five real screenshots, and this README. Ingress was not configured because it was optional; the required deployment and connectivity checks were completed.

The forked GitHub repository is:

`https://github.com/chinmayvyas7/Microservices-Task`

If required by the submission portal, place that URL in a separate `repository-link.txt` file. Push the actual assessment files and screenshots to the fork before submitting the link.
