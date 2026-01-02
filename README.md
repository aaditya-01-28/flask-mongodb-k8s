# Flask + MongoDB on Kubernetes (Minikube)

## Overview
This project demonstrates the deployment of a Python Flask application connected to a MongoDB database on a Kubernetes cluster using Minikube. 

The solution showcases core Kubernetes concepts including Deployments, StatefulSets, Services, Persistent Volumes, Secrets, Autoscaling (HPA), DNS-based service discovery, and resource management. The Flask application exposes REST APIs to insert and retrieve data, while MongoDB is securely deployed with authentication and persistent storage.

## Architecture Flow

```bash
Client (Browser / curl)
        |
   NodePort Service
        |
Flask Deployment (2–5 replicas)
        |
ClusterIP Service (DNS: mongo-service)
        |
MongoDB StatefulSet
        |
Persistent Volume (PVC)
```

## Features Implemented

* Flask REST API with / and /data endpoints
* MongoDB with authentication enabled
* MongoDB deployed as a StatefulSet
* Persistent storage using PVC
* Secure credentials using Kubernetes Secrets
* Internal service-to-service communication via DNS
* Horizontal Pod Autoscaler (HPA) based on CPU usage
* Resource requests and limits for stability

## Prerequisites

Ensure the following are installed on your system:

* Docker
* Minikube
* kubectl
* Python 3.8+ (for local testing only)

Verify installations:
```bash
docker --version
minikube version
kubectl version --client
```

## Project Structure
```bash
flask-mongodb-k8s/
│
├── app.py
├── Dockerfile
├── requirements.txt
├── README.md
│
└── k8s/
    ├── mongo-secret.yaml
    ├── mongo-pvc.yaml
    ├── mongo-statefulset.yaml
    ├── mongo-service.yaml
    ├── flask-deployment.yaml
    ├── flask-service.yaml
    └── flask-hpa.yaml
```
## Flask Application Details
Endpoints
* GET /
Returns a welcome message with the current timestamp.

* POST /data
Inserts JSON data into MongoDB.

* GET /data
Retrieves all stored records from MongoDB.

## Docker Image Build

The Flask application is containerized using Docker.

Dockerfile
```bash
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

Build Image (inside Minikube)
```bash
minikube docker-env | Invoke-Expression   # Windows PowerShell
docker build -t flask-mongo-app:1.0 .
```
## Kubernetes Deployment Steps
1. Start Minikube
```bash
minikube start
```

Enable metrics server (required for HPA):
```bash
minikube addons enable metrics-server
```
2. Deploy MongoDB (with Authentication)

Apply MongoDB resources in order:
```bash
kubectl apply -f k8s/mongo-secret.yaml
kubectl apply -f k8s/mongo-pvc.yaml
kubectl apply -f k8s/mongo-statefulset.yaml
kubectl apply -f k8s/mongo-service.yaml
```

Verify:
```bash
kubectl get pods
kubectl get pvc
kubectl get svc
```
MongoDB pod (mongo-0) should be Running.

3. Deploy Flask Application
```bash
kubectl apply -f k8s/flask-deployment.yaml
kubectl apply -f k8s/flask-service.yaml
```
Verify:
```bash
kubectl get pods
kubectl get svc
```
4. Access the Application

Expose the Flask service:
```bash
minikube service flask-service
```
Or use NodePort directly:
```bash
http://<minikube-ip>:<node-port>
```
5. Enable Autoscaling (HPA)
```bash
kubectl apply -f k8s/flask-hpa.yaml
```

Verify:
```bash
kubectl get hpa
```

## MongoDB Connection (Inside Kubernetes)

MongoDB is accessed using Kubernetes DNS:
```bash
mongo-service
```
MongoDB URI used by Flask:
```bash
mongodb://admin:password@mongo-service:27017/flask_db?authSource=admin
```

* Credentials are stored securely in Kubernetes Secrets
* MongoDB authentication database is admin
* Communication happens entirely inside the cluster

## DNS Resolution in Kubernetes (Explanation)

Kubernetes provides an internal DNS service (CoreDNS).
Every Service automatically receives a DNS name.

In this project:
* MongoDB Service name: mongo-service
* Flask uses this name in the MongoDB URI
```bash
mongodb://admin:password@mongo-service:27017/flask_db
```

When Flask attempts to connect:

1. DNS resolves mongo-service
2. CoreDNS maps it to the MongoDB pod IP
3. Traffic is routed internally within the cluster

This avoids:

* Hardcoded IPs
* External exposure of the database

## Resource Requests and Limits 
**Requests**

* Guaranteed minimum resources
* Used by Kubernetes scheduler

**Limits**

* Maximum allowed usage
* Prevents a pod from exhausting node resources

**Example Used**
```bash
Requests: 0.2 CPU, 250Mi memory
Limits:   0.5 CPU, 500Mi memory
```

**Benefits:**

* Stable performance
* Efficient scheduling
* Enables accurate autoscaling

## Design Choices and Alternatives
**MongoDB as StatefulSet** 
Chosen because:

* Requires stable identity
* Needs persistent storage
**Alternative:** Deployment
**Rejected:** Data loss risk, no stable identity

**Flask as Deployment** 
Chosen because:

* Stateless application
* Easy horizontal scaling

**ClusterIP for MongoDB** 
Chosen because:

* Database should not be exposed externally

**Alternative:** NodePort
**Rejected:** Security risk

**NodePort for Flask** 
Chosen because:

* Simple local access in Minikube

**Alternative:** Ingress

**Rejected:** Overkill for local setup

**Secrets for Credentials** 
Chosen because:

* Secure storage
* Avoids hardcoding credentials

## Cookie Point – Testing Scenarios
Database Testing
```bash
curl -X POST -H "Content-Type: application/json" \
-d '{"name":"test","value":1}' http://<node-ip>:<port>/data

curl http://<node-ip>:<port>/data
```

Data persisted even after MongoDB pod restart, confirming PVC works.

**Autoscaling Testing**
```bash
kubectl run load-generator --image=busybox -- sh
```

Inside pod:
```bash
while true; do wget -q -O- http://flask-service:5000; done
```

Observed:

* CPU usage crossed 70%
* Flask replicas scaled from 2 → 4
* Scaled back down after load stopped

## Autoscaling Results (HPA)

### HPA Before Load
<img width="1818" height="985" alt="hpa-before-load" src="https://github.com/user-attachments/assets/c2a95260-1cab-4406-88f6-10650f6b9d18" />

### Pods Before Load
<img width="1619" height="977" alt="pods-before-load" src="https://github.com/user-attachments/assets/d0c18628-306b-4a2b-be8f-fa5257f56cb0" />

### HPA During High Load
<img width="1489" height="748" alt="hpa-during-load" src="https://github.com/user-attachments/assets/728bbed5-7fc6-480e-a33f-da2df666290b" />

### Pods After Scale-Up
<img width="1572" height="941" alt="pods-after-scale-up" src="https://github.com/user-attachments/assets/36636912-f208-456a-b942-822c4a04d909" />

### Pods Scale Down
<img width="1729" height="1043" alt="pods-scale-down" src="https://github.com/user-attachments/assets/9beea7bf-8b34-4c68-89fa-eba82c3f1299" />

## Issues Encountered & Resolutions

| Issue                 | Resolution                         |
| --------------------- | ---------------------------------- |
| CrashLoopBackOff      | Fixed Flask __name__ == "__main__" |
| MongoDB auth failure  | Added authSource=admin             |
| Service not reachable | Used NodePort IP instead of tunnel |

## Conclusion

This project demonstrates a complete, production-style Kubernetes deployment using best practices.
It covers containerization, secure configuration, service discovery, persistence, autoscaling, and resource management — fulfilling all assignment requirements.
