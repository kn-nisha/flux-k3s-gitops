# Kubernetes GitOps Setup with FluxCD

This is my implementation of a complete Kubernetes deployment using GitOps principles. Everything here is managed declaratively through FluxCD, which automatically syncs the cluster state with this GitHub repository.

## What I Built

I set up a local k3s Kubernetes cluster that runs:
- WordPress with MySQL database
- NGINX Ingress Controller with a custom 404 redirect
- Complete monitoring stack (Prometheus, Loki, Grafana)
- MetalLB for load balancing

FluxCD watches this repository and automatically applies any changes I push.

## How It Works

When I push changes to this repo, FluxCD picks them up within a minute or two and applies them to the cluster. All secrets are encrypted using SOPS with Age encryption, so sensitive data like database passwords are safe to store in the repository.

The infrastructure is organized into layers:
- Infrastructure layer handles the core components (load balancer, ingress)
- Apps layer runs the actual applications (WordPress, MySQL)
- Monitoring layer takes care of observability

Each layer depends on the previous one, so things deploy in the right order automatically.

## Accessing the Applications

Once everything is deployed, you can access:
- WordPress at http://app.local
- Grafana dashboard at http://app-monitor.local

You'll need to add these to your /etc/hosts file, pointing to the MetalLB LoadBalancer IP (mine is 172.21.64.240).

## The 404 Redirect Feature

One requirement was to redirect any 404 errors to Google. I implemented this using a custom Lua plugin in the NGINX Ingress Controller. So if you try to visit http://app.local/notfound or any non-existent path, it redirects you to google.com instead of showing a standard 404 page.

## Monitoring Setup

The monitoring stack collects logs from all pods using Promtail and sends them to Loki for storage. Grafana provides the visualization layer, and I've created a dashboard specifically for viewing WordPress logs. You can see requests, errors, and all application logs in real-time.

Prometheus handles metric collection, though for this assignment, I focused more on the logging side with Loki, as per the requirements.

## Repository Structure

```
flux-k3s-gitops/
├── clusters/local/
│   ├── flux-system/              # FluxCD system components
│   ├── infrastructure/
│   │   ├── metallb/              # Load balancer configs
│   │   ├── metallb-config/       # IP pool configuration
│   │   └── ingress-nginx/        # Ingress controller + Lua plugin
│   ├── apps/
│   │   └── wordpress/            # WordPress and MySQL deployments
│   ├── monitoring/
│   │   ├── prometheus/           # Prometheus + Grafana stack
│   │   └── loki/                 # Loki and Promtail
│   ├── infrastructure-kustomization.yaml
│   ├── metallb-config-kustomization.yaml
│   ├── apps-kustomization.yaml
│   └── monitoring-kustomization.yaml
└── .sops.yaml                    # SOPS encryption config
```

## Secret Management

MySQL credentials are stored encrypted in the repository using SOPS. I generated an Age key pair for encryption, and the private key is stored as a Kubernetes secret that FluxCD uses to decrypt the secrets during deployment.

The encrypted secret file appears as gibberish in the repo, but FluxCD automatically decrypts it when it applies to the cluster. This way, I can safely commit sensitive data to a public repository.

## Technical Details

- **Cluster**: k3s (lightweight Kubernetes)
- **GitOps Tool**: FluxCD v2
- **Secret Encryption**: SOPS with Age
- **Load Balancer**: MetalLB
- **Ingress**: ingress-nginx 4.11.7
- **Monitoring**: Prometheus, Loki, Grafana, Promtail

## Testing It Out

If you want to replicate this setup:

1. Make sure you have k3s installed and running
2. Bootstrap FluxCD pointing to this repository
3. Create the SOPS Age secret in the flux-system namespace
4. Update the MetalLB IP range to match your network
   ```bash
   ip route | grep default  # Find your network
   ```
5. Add entries to /etc/hosts for app.local and app-monitor.local
6. Wait for FluxCD to reconcile (usually 1-2 minutes)

Everything should deploy automatically. You can watch the progress with `flux get kustomizations` and `kubectl get pods -A`.
