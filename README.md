# DevOps Demo: Spring Boot on AWS EKS

End-to-end DevOps project: a Java app built and shipped with CI, provisioned with Infrastructure as Code, deployed with GitOps and monitored with Prometheus/Grafana.

## Architecture (planned)

```
Developer -> GitHub -> GitHub Actions (build, test, Trivy scan, push to ECR)
                              |
                              v
                    Git repo of K8s manifests -> ArgoCD -> AWS EKS
                                                             |
                                       Prometheus + Grafana <-+
Terraform provisions: VPC, EKS, ECR, IAM (state in S3 + DynamoDB)
```

## Tech stack

| Area | Tool |
|------|------|
| App | Java 17, Spring Boot 3, Actuator, Micrometer |
| Container | Docker (multi-stage build) |
| IaC | Terraform |
| CI | GitHub Actions |
| CD / GitOps | ArgoCD + Helm |
| Cloud | AWS (EKS, ECR, VPC, IAM) |
| Monitoring | Prometheus, Grafana |
| Security | Trivy image scan |

## Repo structure

```
app/          Spring Boot application + Dockerfile
terraform/    AWS infrastructure (VPC, EKS, ECR)
k8s/          Kubernetes manifests / Helm chart (watched by ArgoCD)
monitoring/   Prometheus and Grafana configuration
.github/      GitHub Actions workflows
```

## Run the app locally

```bash
cd app
mvn spring-boot:run
# or with Docker
docker build -t devops-demo-app:1.0.0 .
docker run -p 8080:8080 devops-demo-app:1.0.0
```

Endpoints:

- `GET /` returns a hello message and version
- `GET /health` returns `{"status":"UP"}`
- `GET /actuator/prometheus` exposes Prometheus metrics

## Provision the infrastructure (Terraform)

Prerequisites: AWS CLI configured (`aws configure`), Terraform >= 1.6, kubectl.

```bash
# 1. One-time: create the S3 bucket + DynamoDB table for remote state
cd terraform/bootstrap
terraform init
terraform apply -var="state_bucket_name=<your-unique-bucket-name>"

# 2. Create the VPC, EKS cluster and ECR repo
cd ..
cp backend.hcl.example backend.hcl      # edit the bucket name
terraform init -backend-config=backend.hcl
terraform plan
terraform apply

# 3. Connect kubectl (the command is also printed as an output)
aws eks update-kubeconfig --region ap-south-1 --name devops-demo-dev
kubectl get nodes
```

**Cost warning:** EKS (about $0.10/hr) plus a NAT gateway and 2 EC2 nodes add up. Run `terraform destroy` when you finish a session.

## CI pipeline (GitHub Actions)

On every pull request: Maven build and tests, Docker build, Trivy scan (fails on fixable HIGH/CRITICAL vulnerabilities).
On every push to `main`: the same, plus push the image to ECR tagged with the git SHA, then update the image tag in the Helm values file so ArgoCD can deploy it.

One-time setup after `terraform apply`:

1. Copy the `github_ci_role_arn` output.
2. In the GitHub repo go to Settings, Secrets and variables, Actions, and add a secret `AWS_ROLE_ARN` with that value.
3. Optional: add a repository variable `AWS_REGION` if you are not using `ap-south-1`.
4. If your repo is not `Muralik0712/devops-eks-project`, set `github_repo` in Terraform to match.

## Deploy with ArgoCD (GitOps)

The Helm chart lives in `k8s/helm/devops-demo`. ArgoCD watches it in Git, so a deploy is just a commit; CI commits the new image tag after every build.

```bash
# 1. Edit k8s/helm/devops-demo/values.yaml: set image.repository to your ECR URL, then push to GitHub

# 2. Install ArgoCD into the cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Open the ArgoCD UI at https://localhost:8081 (user: admin)
kubectl port-forward svc/argocd-server -n argocd 8081:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 4. Tell ArgoCD to deploy the app
kubectl apply -f k8s/argocd/application.yaml

# 5. Find the public URL of the app
kubectl get svc devops-demo -n devops-demo
```

The repo must be public for ArgoCD to read it without credentials. For a private repo, add it under ArgoCD Settings, Repositories.

## Monitoring (Prometheus + Grafana)

The app exposes metrics at `/actuator/prometheus`. The monitoring stack is installed with Helm, and the app chart adds a ServiceMonitor, alert rules and a Grafana dashboard.

```bash
# 1. Install Prometheus, Alertmanager and Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f monitoring/kube-prometheus-stack-values.yaml

# 2. Turn on the app's monitoring objects: set monitoring.enabled: true in
#    k8s/helm/devops-demo/values.yaml, commit and push. ArgoCD applies it.

# 3. Open Grafana at http://localhost:3000 (user: admin)
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

Alerts: high CPU (over 80% of the limit for 5 minutes), pod restarting (more than 2 restarts in 15 minutes) and target down (2 minutes). The "DevOps Demo App" dashboard shows pods up, request rate, latency, JVM heap, CPU and restarts. Generate some traffic with a loop of `curl` calls against the app's URL to see the graphs move.

## Progress

- [x] Step 1: App and Dockerfile
- [x] Step 2: Terraform (VPC, EKS, ECR)
- [x] Step 3: GitHub Actions pipeline
- [x] Step 4: ArgoCD deployment
- [x] Step 5: Prometheus and Grafana monitoring
- [ ] Step 6: Architecture diagram and final README
