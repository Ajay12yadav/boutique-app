Production-Style CI/CD and GitOps Pipeline for a Microservices App

A DevSecOps pipeline built around Google's Online Boutique demo (10 microservices in Go, Java, C#, Node.js and Python). The application code is upstream. The work in this project is everything around it: the Jenkins CI pipeline, security gates, container registry, Kubernetes manifests, and GitOps delivery with ArgoCD.

Forked from GoogleCloudPlatform/microservices-demo. Application logic is unchanged. Changes are limited to Dockerfiles, dependency/base-image fixes, and the files listed under "What I added".

Architecture
Developer ──push──> GitHub (boutique-app)
                        │
                        ▼
                 Jenkins (Multibranch Pipeline, JCasC)
                   1. Detect changed services (git diff, path-based)
                   2. Gitleaks secret scan
                   3. Unit tests (Go services)
                   4. Docker build
                   5. Trivy scan: OS packages (blocks) + libraries (policy-based)
                   6. Push image to GHCR, tagged with the 7-char commit SHA
                        │
                        ▼
                 GHCR (ghcr.io/ajay12yadav/<service>:<sha>)
                        │
                        ▼
        GitHub (boutique-gitops): Kustomize base + overlays/dev
                        │
                        ▼
                 ArgoCD (watches the repo)
                        │
                        ▼
              Kubernetes (kind) namespace: boutique-dev

Jenkins only does CI. It never runs kubectl against the cluster. ArgoCD pulls the desired state from Git and applies it.

Repositories
Repo	Purpose
boutique-app (this repo)	Application source, Dockerfiles, Jenkinsfile, docker-compose.yml
boutique-gitops	Kubernetes manifests (Kustomize) and ArgoCD Application definitions
jenkins-infra	Jenkins in Docker, plugins list, and Jenkins Configuration as Code (JCasC)
Tech stack
Area	Tools
CI	Jenkins (declarative pipeline, multibranch, JCasC)
Containers	Docker, Docker Compose, multi-stage builds
Security	Gitleaks (secrets), Trivy (OS and library vulnerabilities)
Registry	GitHub Container Registry (GHCR)
Orchestration	Kubernetes (kind), Kustomize
CD	ArgoCD (GitOps)
Local environment	WSL2, 8.7 GB RAM
Services
Service	Language	Tests in pipeline	Library scan policy
frontend	Go	go test	enforce
checkoutservice	Go	go test	enforce
productcatalogservice	Go	go test	enforce
shippingservice	Go	go test	enforce
currencyservice	Node.js	none yet	report
paymentservice	Node.js	none yet	report
emailservice	Python	none yet	report
recommendationservice	Python	none yet	report
adservice	Java	none yet	report
cartservice	C#	none yet	report

redis-cart runs as a plain redis:alpine deployment.

What the pipeline does
1. Path-based builds (monorepo)

The Detect changes stage runs git diff --name-only against the last successful build and builds only the services whose folders changed. It falls back to building everything when:

there is no previous successful build,
the diff fails,
the Jenkinsfile itself changed,
the BUILD_ALL parameter is ticked.

A docs-only commit builds nothing.

2. Security gates
Gitleaks scans the workspace for committed secrets.
Trivy runs two scans per image:
OS packages (base image): always blocks the build on fixable HIGH/CRITICAL findings.
Application libraries (npm, pip, etc.): blocks for services set to enforce. For upstream demo services set to report, findings are printed but do not fail the build (see "Security exceptions").
Trivy's vulnerability database is cached in a Docker volume (trivy-cache) and scans use --skip-db-update, so builds do not depend on a slow network.
3. Registry and tagging

Images are pushed to GHCR only from the main branch, tagged with the 7-character commit SHA. Registry credentials come from the Jenkins credential store and are never printed to the log (single-quoted sh blocks, docker logout after push).

4. GitOps delivery

Image tags live in boutique-gitops/overlays/dev/kustomization.yaml. ArgoCD watches that repo with automated sync, prune and selfHeal enabled. Changing a tag in Git is how a new version is deployed, and git revert is how it is rolled back.

Run it locally
Docker Compose (no Kubernetes)
bash
git clone https://github.com/Ajay12yadav/boutique-app.git
cd boutique-app
docker compose up -d --build
# open http://localhost:8080
docker compose down
Jenkins
bash
cd jenkins-infra
cp .env.example .env        # set JENKINS_ADMIN_PASSWORD and DOCKER_GID
docker compose up -d --build
# open http://localhost:8090

Jenkins is configured from casc/jenkins.yaml, so the setup is reproducible. Credentials (github-pat, ghcr-creds) are added through the Jenkins credentials store.

Kubernetes and ArgoCD
bash
kind create cluster --name boutique
kubectl config current-context          # must print kind-boutique

kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl apply -f argocd/dev-app.yaml    # from the boutique-gitops repo
kubectl port-forward -n boutique-dev svc/frontend 8080:80
Security exceptions
Service	Finding	Why accepted	Revisit
currencyservice, paymentservice and other upstream-dependency services	HIGH/CRITICAL npm findings in transitive dependencies (for example protobufjs, lodash)	Upstream demo code. Most vulnerable packages come in through the profiler and tracing libraries, which are disabled in this deployment. Overriding them risked breaking the services.	Before any real-world use

The library scan still runs and prints these findings on every build, so nothing is hidden. OS-level findings are never excepted.

Problems I diagnosed and fixed
Problem	Symptom	Cause	Fix
Compose file not found	no configuration file provided	File was named Docker-Compose.yaml, and Linux is case-sensitive	Renamed to docker-compose.yml
npm install timeout	ETIMEDOUT in node-gyp for paymentservice and currencyservice	The optional pprof profiler tried to compile and download Node headers	Disabled the profiler, changed install to npm install --omit=dev --ignore-scripts
Frontend crash loop	panic: environment variable "SHOPPING_ASSISTANT_SERVICE_ADDR" not set	Upstream frontend requires a new variable at startup	Added the variable to the Compose file and manifests
Jenkinsfile would not parse	unexpected char: ''`	Markdown code fences were copied into the file	Removed the fence lines
go test failed on checkoutservice	non-constant format string in call to status.Errorf	Stricter go vet in newer Go versions	Used a constant format string in the call
Trivy database download stalled	Scan hung at 0.17% of the 120 MB DB download	Every scan started with an empty cache	Persistent trivy-cache volume and --skip-db-update
Trivy blocked on OpenSSL	libcrypto3/libssl3 HIGH CVE on Alpine, OpenSSL HIGH CVE on Ubuntu	Pinned base images lagged behind patched packages	Upgraded OS packages in the runtime stage, or moved to a patched base tag
ImagePullBackOff on Kubernetes	not found for the image tag	Kustomize had no newTag, and the tag used did not exist in GHCR	Per-service tags taken from the real GHCR tag list
CrashLoopBackOff on Python services	Liveness probe failed ... within 1s, exit code 137	Probe timeout and delay too strict for a busy node	Longer timeoutSeconds/initialDelaySeconds and a startupProbe
Project status
Stage	Status
Containerize all services, run with Docker Compose	Done
Jenkins in Docker with JCasC	Done
Multibranch pipeline, path-based builds, Gitleaks, Trivy, GHCR push	Done
Kubernetes manifests with Kustomize on kind	Done
ArgoCD GitOps with automated sync, prune and self-heal	Done
Jenkins stage that commits new image tags to the GitOps repo	Planned
GitHub webhook trigger (ngrok)	Planned
Jenkins Shared Library	Planned
dev / staging / prod promotion with manual approval	Planned
Argo Rollouts canary with Prometheus-based rollback	Planned
Prometheus, Grafana and alerting	Planned
SonarQube quality gate	Planned
Cosign image signing, Kyverno policies, k6 load test	Planned
Terraform for cloud infrastructure	Planned
What I added to the upstream project
Jenkinsfile (path-based multi-service pipeline)
docker-compose.yml (local environment for all services)
Dockerfile fixes for paymentservice, currencyservice, cartservice and other services where Trivy found OS-level issues
docs/service-inventory.md (language, port, Dockerfile path and test method per service)
Companion repos boutique-gitops and jenkins-infra
Screenshots




Application code is licensed under Apache 2.0 by Google LLC (see upstream). Additions in this repository are provided for learning purposes.
  showing Stackdriver Incident Response Management
- [Microservices demo showcasing Go Micro](https://github.com/go-micro/demo)
