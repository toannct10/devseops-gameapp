# DevSecOps Pipeline Implementation for Tic Tac Toe Game

![Screenshot 2025-03-04 at 7 16 48 PM](https://github.com/user-attachments/assets/7ed79f9c-9144-4870-accd-500085a15592)

![image](https://github.com/user-attachments/assets/5b2813a5-f493-4665-8964-77359b5be93a)

## Features

- 🎮 Fully functional Tic Tac Toe game
- 📊 Score tracking for X, O, and draws
- 📜 Game history with timestamps
- 🏆 Highlights winning combinations
- 🔄 Reset game and statistics
- 📱 Responsive design for all devices

## Technologies Used

- React 18
- TypeScript
- Tailwind CSS
- Lucide React for icons

## Project Structure

```
src/
├── components/
│   ├── Board.tsx       # Game board component
│   ├── Square.tsx      # Individual square component
│   ├── ScoreBoard.tsx  # Score tracking component
│   └── GameHistory.tsx # Game history component
├── utils/
│   └── gameLogic.ts    # Game logic utilities
├── App.tsx             # Main application component
└── main.tsx           # Entry point
```

## Game Logic

The game implements the following rules:

1. X goes first, followed by O
2. The first player to get 3 of their marks in a row (horizontally, vertically, or diagonally) wins
3. If all 9 squares are filled and no player has 3 marks in a row, the game is a draw
4. Winning combinations are highlighted
5. Game statistics are tracked and displayed

## Getting Started

### Prerequisites

- Node.js (v14 or higher)
- npm or yarn

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/devsecops-demo.git
   cd devsecops-demo
   ```

2. Install dependencies:
   ```bash
   npm install
   # or
   yarn
   ```

3. Start the development server:
   ```bash
   npm run dev
   # or
   yarn dev
   ```

4. Open your browser and navigate to `http://localhost:5173`

## Building for Production

To create a production build:

```bash
npm run build
# or
yarn build
```

The build artifacts will be stored in the `dist/` directory.

## Continuous Deployment (CD) – EC2, Kind & Argo CD

This section describes how to deploy the application to a Kubernetes cluster running on an AWS EC2 instance using Kind and Argo CD.

### 1. Create EC2 Instance

- Create an EC2 instance on AWS.
- Instance type: `t2.medium`
- OS: Ubuntu Server
- Configure Security Group:
  - SSH: TCP `22` – My IP
  - Argo CD UI: TCP `9000` – My IP

### 2. Connect to EC2

Connect to your EC2 instance using SSH:

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

### 3. Install Docker

Update packages and install Docker:

```bash
sudo apt update
sudo apt install docker.io -y
```

Add the Ubuntu user to the Docker group:

```bash
sudo usermod -aG docker ubuntu
```

Log out and reconnect to apply the group permissions, then verify Docker:

```bash
docker --version
docker ps
```

Login to GitHub Container Registry (GHCR):

```bash
docker login ghcr.io -u YOUR_GITHUB_USERNAME
```

Enter your GitHub Personal Access Token (PAT) with `read:packages` permission when prompted.

### 4. Install Kind (Kubernetes in Docker)

Download and install Kind:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verify installation and create a Kubernetes cluster:

```bash
kind version
kind create cluster --name devops-demo-cluster
```

Check the cluster:

```bash
kubectl cluster-info
```

### 5. Install kubectl

Download the latest stable kubectl binary:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install kubectl:

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Verify the installation and current context:

```bash
kubectl version --client
kubectl config current-context
kubectl get nodes
```

### 6. Install Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd --server-side --force-conflicts \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for Argo CD pods to become ready:

```bash
kubectl get pods -n argocd -w
```

### 7. Access Argo CD Web UI

Expose the Argo CD server on port `9000`:

```bash
kubectl port-forward svc/argocd-server 9000:80 \
-n argocd --address 0.0.0.0
```

Open the browser:

```text
http://<EC2_PUBLIC_IP>:9000
```

### 8. Get Initial Admin Password

Retrieve the initial Argo CD admin password:

```bash
kubectl get secret argocd-initial-admin-secret \
-n argocd \
-o jsonpath="{.data.password}" | base64 -d
```

Login to Argo CD:

- Username: `admin`
- Password: Output from the previous command.

### 9. Create GHCR Image Pull Secret

Create a Kubernetes Docker Registry Secret so that Kubernetes can pull private images from GitHub Container Registry.

```bash
kubectl create secret docker-registry github-container-registry \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_TOKEN \
  --docker-email=YOUR_EMAIL
```

The GitHub token must have `read:packages` permission.

Verify the secret:

```bash
kubectl get secret github-container-registry
```

**Important:** Reference this secret in your Kubernetes Deployment manifest:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: github-container-registry
      containers:
        - name: app
          image: ghcr.io/YOUR_GITHUB_USERNAME/YOUR_IMAGE:latest
```

### 10. Configure Argo CD Application

Create an Argo CD Application that points to your GitHub repository and Kubernetes deployment manifests.

Example configuration:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devops-demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git
    targetRevision: main
    path: kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply the Application:

```bash
kubectl apply -f argocd/application.yaml
```

### 11. Verify Deployment

Check application resources:

```bash
kubectl get applications -n argocd
kubectl get pods -A
kubectl get deployments
kubectl get services
```

Once Argo CD synchronizes the manifests from GitHub, the application will be deployed to the Kind Kubernetes cluster.

### Continuous Deployment Workflow

1. Developer pushes code changes to GitHub.
2. GitHub Actions builds and scans the Docker image.
3. GitHub Actions pushes the image to GHCR.
4. Argo CD detects changes in the Git repository.
5. Argo CD synchronizes Kubernetes manifests.
6. Kubernetes pulls the new image from GHCR and updates the application.

**Architecture:**

`GitHub → GitHub Actions → GHCR → Argo CD → Kind Kubernetes → Application`

