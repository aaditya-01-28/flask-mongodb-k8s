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

## DNS Resolution in Kubernetes

* Kubernetes provides an internal DNS service (CoreDNS)
* Each Service gets a DNS name
* Flask connects to MongoDB using the service name mongo-service
* DNS resolves this name to the MongoDB pod IP dynamically
* No hardcoded IPs are used

## Resource Requests and Limits

Both Flask and MongoDB pods define:

* Requests: Guaranteed minimum resources
* Limits: Maximum allowed usage

Example:
```bash
Requests: 0.2 CPU, 250Mi memory
Limits:   0.5 CPU, 500Mi memory
```

This ensures:

* Stable scheduling
* Fair resource usage
* Proper autoscaling behavior

## Autoscaling Test (Cookie Point)

To simulate load:
```bash
kubectl run load-generator --image=busybox -- sh
```

Inside the pod:
```bash
while true; do wget -q -O- http://flask-service:5000; done
```

Observe scaling:
```bash
kubectl get hpa
kubectl get pods
```
Flask replicas increase when CPU usage exceeds 70%.

## Design Choices

* **StatefulSet for MongoDB**: Required for stable identity and storage
* **Deployment for Flask:** Stateless application, easy scaling
* **ClusterIP for MongoDB:** Database should not be exposed externally
* **NodePort for Flask:** Simple local access in Minikube
* **Secrets for credentials:** Avoid hardcoding sensitive data
* **PVC for MongoDB:** Ensures data persistence across restarts


## Issues Encountered & Resolutions

**Issue**	                    **Resolution**
CrashLoopBackOff	        Fixed Flask __name__ == "__main__"
MongoDB auth failure	    Added authSource=admin
Service not reachable	Used NodePort IP instead of tunnel

## Conclusion

This project demonstrates a complete, production-style Kubernetes deployment using best practices.
It covers containerization, secure configuration, service discovery, persistence, autoscaling, and resource management — fulfilling all assignment requirements.