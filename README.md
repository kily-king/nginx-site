
# ArgoCD_Nginx_CICD_StepByStep

Argo CD + GitHub Actions CI/CD for Nginx — Step-by-Step Commands & YAMLs
This document contains a complete, ordered list of commands and all YAML/files needed. Copy-paste each command on your VM.
Prerequisites (VM & Kubernetes)
kubectl configured to target your Kubernetes cluster
git installed and SSH key for GitHub configured (optional)
You have access to create GitHub repos and GitHub Actions (or create account at https://github.com).
1) Create GitHub repo (on your machine via gh CLI or via web)
Web: create repo named `nginx-site` under your account.
Optional CLI (gh):
gh repo create nginx-site --public --source=. --remote=origin --push
2) Clone repo to VM
git clone git@github.com:<your-github-username>/nginx-site.git
cd nginx-site
3) Create project folder structure (one-shot)
mkdir -p nginx-site/html nginx-site/k8s nginx-site/argocd && mkdir -p nginx-site/.github/workflows
cd nginx-site
4) Create html/index.html
cat > html/index.html <<'EOF'
<!DOCTYPE html>
<html>
  <head><meta charset="UTF-8"><title>Nginx Site via Argo CD</title></head>
  <body>
    <h1>Hello from Nginx on Kubernetes!</h1>
    <p>Deployed using GitHub Actions + Argo CD.</p>
  </body>
</html>
EOF
5) Create Dockerfile
cat > Dockerfile <<'EOF'
FROM nginx:latest
RUN rm -rf /usr/share/nginx/html/*
COPY html/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
6) Create k8s/deployment.yaml
cat > k8s/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-site
  labels:
    app: nginx-site
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-site
  template:
    metadata:
      labels:
        app: nginx-site
    spec:
      containers:
      - name: nginx
        image: ghcr.io/<your-github-username>/nginx-site:latest
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
EOF
7) Create k8s/service.yaml
cat > k8s/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-site-svc
spec:
  type: ClusterIP
  selector:
    app: nginx-site
  ports:
  - name: http
    port: 80
    targetPort: 80
EOF
8) Create Argo CD Application manifest (argocd/app-nginx-site.yaml)
cat > argocd/app-nginx-site.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-site
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<your-github-username>/nginx-site.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
9) Create GitHub Actions workflow (.github/workflows/ci.yml)
cat > .github/workflows/ci.yml <<'EOF'
name: CI for Nginx Website

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    env:
      IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/nginx-site:latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Docker image
        run: |
          docker build -t $IMAGE_NAME .

      - name: Push Docker image
        run: |
          docker push $IMAGE_NAME
EOF
10) Commit & push files to GitHub
git add .
git commit -m "Initial nginx-site CI/CD files"
git push origin main




Kubernetes SetUp

11) Install Argo CD in cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
12) Install argocd CLI (on VM where you run commands)
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
13) Login to Argo CD CLI
argocd login localhost:8080 --username admin --password <PASTE_PASSWORD_FROM_SECRET> --insecure
14) Add Git repo to Argo CD (SSH recommended)
ssh-keygen -t ed25519 -f ~/.ssh/argocd_id_ed25519 -N "" || true
cat ~/.ssh/argocd_id_ed25519.pub  # copy and add as Deploy Key (read-only) in GitHub repo settings
argocd repo add git@github.com:<your-github-username>/nginx-site.git --ssh-private-key-path ~/.ssh/argocd_id_ed25519 --insecure-ignore-host-key
15) Apply Argo CD Application manifest
kubectl apply -f argocd/app-nginx-site.yaml
16) Sync application (if automated syncPolicy is set it will auto-sync after repo update)
argocd app sync nginx-site
argocd app get nginx-site
17) Access service locally
kubectl port-forward svc/nginx-site-svc 8080:80
Open browser: http://localhost:8080
18) Optional: Change Service to LoadBalancer (cloud) or configure Ingress
# Edit k8s/service.yaml -> change type: LoadBalancer
kubectl apply -f k8s/service.yaml
19) Troubleshooting tips (short)
- Check pods: kubectl get pods -l app=nginx-site
- Check events: kubectl describe pod <pod-name>
- ArgoCD logs: kubectl -n argocd logs deploy/argocd-application-controller
- Workflow failure: view Actions tab in GitHub repo



# Old Commands

ubuntu@ip-172-31-20-166:~$ history
    1  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    2  sudo apt install unzip
    3  sudo unzip awscliv2.zip
    4  sudo ./aws/install
    5  aws --version
    6  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    7  sudo mv /tmp/eksctl /usr/local/bin
    8  eksctl version
    9  sudo curl --silent --location -o /usr/local/bin/kubectl   https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl
   10  sudo chmod +x /usr/local/bin/kubectl
   11  kubectl version --short --client
   12  history
IAM Roles

eksctl create cluster --name demo-eks --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --managed --nodes 2

ubuntu@ip-172-31-26-115:~$ history
    1  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    2  sudo apt install unzip
    3  sudo unzip awscliv2.zip
    4  sudo ./aws/install
    5  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    6  sudo mv /tmp/eksctl /usr/local/bin
    7  sudo curl --silent --location -o /usr/local/bin/kubectl   https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl
    8  sudo chmod +x /usr/local/bin/kubectl
    9  kubectl version --short --client
   10  eksctl create cluster --name demo-eks --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --managed --nodes 2
   11  eksctl get cluster --name demo-eks --region us-east-1
   12  aws eks update-kubeconfig --name demo-eks --region us-east-1
   13  cat  /var/lib/jenkins/.kube/config
   14  kubectl get nodes
   15  kubectl get ns
   16  kubectl create deployment nginx --image=nginx
   17  kubectl get deploymentskubectl get deployments
   18  kubectl get deployments
   19  helm repo add stable https://charts.helm.sh/stable
   20  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
   21  sudo chmod 700 get_helm.sh
   22  sudo ./get_helm.sh
   23  helm version --client
   24  helm repo add stable https://charts.helm.sh/stable
   25  free -h
   26  helm repo add stable https://charts.helm.sh/stable
   27  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   28  helm search repo prometheus-community
   29  kubectl create namespace prometheus
   30  helm install stable prometheus-community/kube-prometheus-stack -n prometheus
   31  kubectl get pods -n prometheus
   32  kubectl get svc -n prometheus
   33  kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
   34  kubectl edit svc stable-grafana -n prometheus
   35  kubectl get svc -n prometheus
   36  kubectl get secret --namespace prometheus -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echokubectl get secret --namespace prometheus -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
   37  kubectl get secret --namespace prometheus -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
   38  history






