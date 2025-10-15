[devops_ci_cd_kubernetes_repository_scaffold.md](https://github.com/user-attachments/files/22917837/devops_ci_cd_kubernetes_repository_scaffold.md)
# devops-ci-cd-kubernetes

 proyecto DevOps: CI/CD con GitHub Actions y despliegue en Kubernetes.

---

## Árbol de archivos

```
devops-ci-cd-kubernetes/
│
├── app/
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
│
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
│
├── .dockerignore
├── README.md
```

---

## Contenido de los archivos

### `app/main.py`

```python
from flask import Flask, jsonify
import os

app = Flask(_haroldapp_)

@app.route('/')
def hello():
    version = os.getenv('APP_VERSION', 'v1')
    return jsonify({
        'message': 'respuesta DevOps CI/CD app',
        'version': version
    })

if _haroldapp_ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

### `app/requirements.txt`

```
Flask==2.2.5
```

---

### `app/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV APP_VERSION=local
EXPOSE 5000
CMD ["python", "main.py"]
```

---

### `.dockerignore`

```
__pycache__
*.pyc
*.pyo
*.pyd
.env
venv
.git
```

---

### `k8s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-ci-cd
```

---

### `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
  namespace: devops-ci-cd
  labels:
    app: devops-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-app
  template:
    metadata:
      labels:
        app: devops-app
    spec:
      containers:
        - name: devops-app
          image: yourdockerhubuser/devops-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 5000
          env:
            - name: APP_VERSION
              value: "${GIT_TAG:-latest}"
```

---

### `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: devops-app-service
  namespace: devops-ci-cd
spec:
  selector:
    app: devops-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: LoadBalancer
```



---

### `.github/workflows/ci-cd.yaml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  IMAGE_NAME: devops-app

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        id: build
        run: |
          IMAGE_TAG=${{ secrets.DOCKERHUB_USERNAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker build -t $IMAGE_TAG ./app
          docker push $IMAGE_TAG
          echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Create kubeconfig
        if: ${{ secrets.KUBECONFIG }}
        run: |
          echo "${{ secrets.KUBECONFIG }}" > kubeconfig
          mkdir -p $HOME/.kube
          mv kubeconfig $HOME/.kube/config

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'

      - name: Deploy to Kubernetes
        env:
          IMAGE_TAG: ${{ steps.build.outputs.IMAGE_TAG }}
        run: |
          # Reemplaza la imagen en el deployment con la nueva TAG
          kubectl -n devops-ci-cd apply -f k8s/namespace.yaml || true
          kubectl -n devops-ci-cd set image deployment/devops-app devops-app=$IMAGE_TAG --record || true
          kubectl -n devops-ci-cd apply -f k8s/deployment.yaml
          kubectl -n devops-ci-cd apply -f k8s/service.yaml

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run unit tests
        run: |
          echo "No tests defined. Add pytest or similar and run here."


