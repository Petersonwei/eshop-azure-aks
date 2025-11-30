# EShop Modular Monolith - AKS Deployment Guide 
(November 2025)

This guide provides step-by-step instructions to deploy the EShop modular monolith application to Azure Kubernetes Service (AKS).

## Prerequisites

- Azure CLI installed and logged in
- Docker installed and running
- kubectl installed
- Azure subscription with permissions to create resources

## Project Overview

This is a .NET 8 modular monolith e-commerce application with the following components:
- **Main API**: ASP.NET Core Web API (`Bootstrapper/Api`)
- **Database**: PostgreSQL with identity schema
- **Cache**: Redis
- **Message Broker**: RabbitMQ
- **Identity**: Keycloak
- **Logging**: Seq

## Step 1: Create Azure Resources

### 1.1 Create Resource Group
```bash
az group create --name eshop-rg --location eastus
```

### 1.2 Create Azure Container Registry (ACR)
```bash
az acr create --resource-group eshop-rg --name eshopacr$(date +%s) --sku Standard --location eastus
```

Get your ACR name:
```bash
ACR_NAME=$(az acr list --resource-group eshop-rg --query "[0].name" -o tsv)
echo $ACR_NAME
```

### 1.3 Create AKS Cluster
```bash
az aks create \
    --resource-group eshop-rg \
    --name eshop-aks \
    --node-count 3 \
    --node-vm-size Standard_DS2_v2 \
    --attach-acr $ACR_NAME \
    --generate-ssh-keys \
    --enable-managed-identity \
    --location eastus
```

### 1.4 Get AKS Credentials
```bash
az aks get-credentials --resource-group eshop-rg --name eshop-aks
```

## Step 2: Build and Push Docker Images

### 2.1 Login to ACR
```bash
az acr login --name $ACR_NAME
ACR_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
```

### 2.2 Build and Push Main API Image
```bash
# Build the image
docker build -f Bootstrapper/Api/Dockerfile -t $ACR_SERVER/eshop-api:latest .

# Push to ACR
docker push $ACR_SERVER/eshop-api:latest
```

## Step 3: Create Kubernetes Manifests

### 3.1 Create Namespace
Create `k8s/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: eshop
  labels:
    name: eshop
```

### 3.2 Create ConfigMap for Database Init
Create `k8s/postgres-init-configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init-config
  namespace: eshop
data:
  init-db.sql: |
    -- Initialize required schemas for the application
    CREATE SCHEMA IF NOT EXISTS identity;
```

### 3.3 Create PostgreSQL Deployment
Create `k8s/postgres.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: eshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: postgres
        - name: POSTGRES_DB
          value: EShopDb
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: init-script
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: postgres-storage
        emptyDir: {}
      - name: init-script
        configMap:
          name: postgres-init-config
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: eshop
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### 3.4 Create Redis Deployment
Create `k8s/redis.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: eshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: eshop
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

### 3.5 Create RabbitMQ Deployment
Create `k8s/rabbitmq.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq
  namespace: eshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3-management
        env:
        - name: RABBITMQ_DEFAULT_USER
          value: guest
        - name: RABBITMQ_DEFAULT_PASS
          value: guest
        ports:
        - containerPort: 5672
        - containerPort: 15672
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  namespace: eshop
spec:
  selector:
    app: rabbitmq
  ports:
  - name: amqp
    port: 5672
    targetPort: 5672
  - name: management
    port: 15672
    targetPort: 15672
```

### 3.6 Create Keycloak Deployment
Create `k8s/keycloak.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: eshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:24.0.3
        env:
        - name: KEYCLOAK_ADMIN
          value: admin
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: admin
        - name: KC_DB
          value: postgres
        - name: KC_DB_URL
          value: jdbc:postgresql://postgres/EShopDb?currentSchema=identity
        - name: KC_DB_USERNAME
          value: postgres
        - name: KC_DB_PASSWORD
          value: postgres
        command:
        - /opt/keycloak/bin/kc.sh
        - start-dev
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: eshop
spec:
  selector:
    app: keycloak
  ports:
  - port: 8080
    targetPort: 8080
```

### 3.7 Create Seq Deployment
Create `k8s/seq.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seq
  namespace: eshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: seq
  template:
    metadata:
      labels:
        app: seq
    spec:
      containers:
      - name: seq
        image: datalust/seq:latest
        env:
        - name: ACCEPT_EULA
          value: "Y"
        - name: SEQ_FIRSTRUN_ADMINPASSWORD
          value: admin
        ports:
        - containerPort: 5341
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: seq
  namespace: eshop
spec:
  selector:
    app: seq
  ports:
  - name: api
    port: 5341
    targetPort: 5341
  - name: web
    port: 80
    targetPort: 80
```

### 3.8 Create Main API ConfigMap
Create `k8s/api-configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: eshop
data:
  appsettings.json: |
    {
      "ConnectionStrings": {
        "Database": "Server=postgres;Port=5432;Database=EShopDb;User Id=postgres;Password=postgres;Include Error Detail=true",
        "Redis": "redis:6379"
      },
      "MessageBroker": {
        "Host": "amqp://rabbitmq:5672",
        "UserName": "guest",
        "Password": "guest"
      },
      "Keycloak": {
        "realm": "myrealm",
        "auth-server-url": "http://keycloak:8080/",
        "ssl-required": "none",
        "resource": "myclient",
        "verify-token-audience": false,
        "credentials": {
          "secret": ""
        },
        "confidential-port": 0
      },
      "Serilog": {
        "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.Seq" ],
        "MinimumLevel": {
          "Default": "Information",
          "Override": {
            "Microsoft": "Information",
            "System": "Warning"
          }
        },
        "WriteTo": [
          {
            "Name": "Console"
          },
          {
            "Name": "Seq",
            "Args": {
              "serverUrl": "http://seq:5341"
            }
          }
        ],
        "Enrich": [ "FromLogContext", "WithMachineName", "WithProcessId", "WithThreadId" ],
        "Properties": {
          "Application": "EShop ASP.NET Core App",
          "Environment": "Production"
        }
      },
      "AllowedHosts": "*"
    }
```

### 3.9 Create Main API Deployment
Create `k8s/api.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eshop-api
  namespace: eshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eshop-api
  template:
    metadata:
      labels:
        app: eshop-api
    spec:
      containers:
      - name: eshop-api
        image: YOUR_ACR_SERVER/eshop-api:latest  # Replace with your ACR server
        ports:
        - containerPort: 8080
        - containerPort: 8081
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ASPNETCORE_URLS
          value: "http://+:8080"
        volumeMounts:
        - name: config
          mountPath: /app/appsettings.json
          subPath: appsettings.json
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: api-config
---
apiVersion: v1
kind: Service
metadata:
  name: eshop-api
  namespace: eshop
spec:
  selector:
    app: eshop-api
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### 3.10 Create Ingress (Optional)
Create `k8s/ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: eshop-ingress
  namespace: eshop
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: eshop.yourdomain.com  # Replace with your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: eshop-api
            port:
              number: 80
```

## Step 4: Deploy to AKS

### 4.1 Create the k8s directory and files
```bash
mkdir -p k8s
# Create all the YAML files above in the k8s directory
```

### 4.2 Update the API deployment image
First, get your ACR server name:
```bash
echo $ACR_SERVER
```

Then update the `k8s/api.yaml` file and replace `YOUR_ACR_SERVER` with your actual ACR server name.

### 4.3 Deploy all components
```bash
# Apply namespace
kubectl apply -f k8s/namespace.yaml

# Apply database and dependencies first
kubectl apply -f k8s/postgres-init-configmap.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/redis.yaml
kubectl apply -f k8s/rabbitmq.yaml
kubectl apply -f k8s/seq.yaml

# Wait for database to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n eshop --timeout=300s

# Deploy Keycloak
kubectl apply -f k8s/keycloak.yaml

# Wait for Keycloak to be ready
kubectl wait --for=condition=ready pod -l app=keycloak -n eshop --timeout=300s

# Deploy the main API
kubectl apply -f k8s/api-configmap.yaml
kubectl apply -f k8s/api.yaml

# Optional: Apply ingress
# kubectl apply -f k8s/ingress.yaml
```

## Step 5: Verify Deployment

### 5.1 Check pod status
```bash
kubectl get pods -n eshop
```

### 5.2 Check services
```bash
kubectl get services -n eshop
```

### 5.3 Get external IP for the API
```bash
kubectl get service eshop-api -n eshop
```

Wait for the EXTERNAL-IP to be assigned. This may take a few minutes.

### 5.4 Test the application
Once you have the external IP, you can access:
- Main API: `http://EXTERNAL-IP/swagger`
- RabbitMQ Management: `http://EXTERNAL-IP:15672` (guest/guest)
- Seq Logs: `http://EXTERNAL-IP:80` (for seq service)
- Keycloak: `http://EXTERNAL-IP:8080` (admin/admin)

## Step 6: Monitoring and Troubleshooting

### 6.1 View logs
```bash
# API logs
kubectl logs -f deployment/eshop-api -n eshop

# Database logs
kubectl logs -f deployment/postgres -n eshop

# All pods logs
kubectl logs -f -l app=eshop-api -n eshop
```

### 6.2 Debug pod issues
```bash
# Describe a pod
kubectl describe pod POD_NAME -n eshop

# Get into a pod shell
kubectl exec -it POD_NAME -n eshop -- /bin/bash
```

### 6.3 Check resource usage
```bash
kubectl top pods -n eshop
kubectl top nodes
```

## Step 7: Scaling and Updates

### 7.1 Scale the API
```bash
kubectl scale deployment eshop-api --replicas=3 -n eshop
```

### 7.2 Update the application
```bash
# Build new version
docker build -f Bootstrapper/Api/Dockerfile -t $ACR_SERVER/eshop-api:v1.1 .
docker push $ACR_SERVER/eshop-api:v1.1

# Update deployment
kubectl set image deployment/eshop-api eshop-api=$ACR_SERVER/eshop-api:v1.1 -n eshop

# Check rollout status
kubectl rollout status deployment/eshop-api -n eshop
```

## Step 8: Production Considerations

### 8.1 Use Azure Database for PostgreSQL
For production, consider using Azure Database for PostgreSQL instead of containerized PostgreSQL:

```bash
az postgres flexible-server create \
    --resource-group eshop-rg \
    --name eshop-postgres \
    --admin-user adminuser \
    --admin-password YourStrongPassword123! \
    --sku-name Standard_B2s \
    --storage-size 32 \
    --location eastus
```

### 8.2 Use Azure Cache for Redis
```bash
az redis create \
    --resource-group eshop-rg \
    --name eshop-redis \
    --location eastus \
    --sku Basic \
    --vm-size c0
```

### 8.3 Enable SSL/TLS
- Use Azure Application Gateway for SSL termination
- Configure proper certificates
- Update ingress configuration

### 8.4 Secrets Management
Use Azure Key Vault and AKS secrets store CSI driver for managing secrets securely.

## Cleanup

To remove all resources:
```bash
# Delete AKS deployment
kubectl delete namespace eshop

# Delete Azure resources
az group delete --name eshop-rg --yes --no-wait
```

## Notes

- This deployment uses basic authentication and is suitable for development/testing
- For production, implement proper security measures including secrets management, network policies, and RBAC
- Monitor resource usage and adjust requests/limits accordingly
- Consider using Helm charts for easier management
- Set up proper backup strategies for persistent data

The application should now be accessible via the LoadBalancer external IP. Check the Swagger UI to verify the API is working correctly.