## Architecture Overview

This project is built around **YOURLS** (a self-hosted URL shortener) and **MariaDB** (the relational database required by YOURLS). 

The infrastructure is split into two distinct namespaces representing different environments: `test-environment` and `production-ready`.

### 1. Test Environment (`test-environment`)
This namespace is designed for local development and quick validation.
* **Database:** Launches a single replica of MariaDB. Configuration of custom resources (`mariadbs`, `databases`, `users`, `grants`) is automated.
* **Application:** Launches YOURLS as a single replica using baseline configurations.
* **Secrets Management:** Secrets for the MariaDB root/user access and the YOURLS admin dashboard are automatically generated and injected via **Kustomize**.
* **Networking:** Local routing to the application is managed via a Traefik **IngressRoute** custom resource, mapped to the local hostname `yourls.local`.

### 2. Production Environment (`production-ready`)
This namespace is built for high availability, security, and external access.
* **Database High Availability:** Launches 3 replicas of MariaDB configured in **Galera Cluster mode** for synchronous data replication across nodes. *(Note: For this specific deployment, nodes run as separate pods on a single physical host node).*
* **Application Scaling:** YOURLS scales dynamically from **2 to 5 replicas** using a Horizontal Pod Autoscaler (HPA), driven by defined CPU/Memory resource requests and limits.
* **GitOps Security:** Sensitive configuration files and secrets are encrypted using **SOPS** and **age**, allowing them to be safely committed and pushed to GitHub.
* **Secure External Access:** Inbound traffic is proxied securely via a **Cloudflare Tunnel (`cloudflared`)** running between Cloudflare's edge and the local cluster. 

#### 🔒 Reverse Proxy & HTTPS Termination Note
HTTPS encryption terminates at the Cloudflare Edge network. The `cloudflared` tunnel forwards this traffic to the local Traefik service, which handles internal routing to YOURLS. 

Because the internal cluster traffic travels over HTTP, Traefik must explicitly pass the `X-Forwarded-Proto: https` header down to the YOURLS pods. This is handled via a Traefik **Middleware** custom resource. Without this header, YOURLS suffers from a **Mixed Content Mismatch error**, causing the URL creation AJAX requests to spin indefinitely in the browser.
