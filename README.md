# Kubernetes Infrastructure

Production-ready Kubernetes manifests for blockchain RPC nodes with full observability stack.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Ingress                              │
│                       (NGINX)                               │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌───────────┐  ┌───────────┐  ┌───────────┐
│  Grafana  │  │ Solana    │  │   ...     │
│           │  │ RPC       │  │           │
└─────┬─────┘  └─────┬─────┘  └───────────┘
      │              │
      ▼              │
┌───────────┐        │
│Prometheus │◄───────┘ (metrics)
└─────┬─────┘
      │
      ▼
┌───────────┐
│   Loki    │◄──────── (logs via Promtail)
└───────────┘
```

## Components

| Component | Description | Namespace |
|-----------|-------------|-----------|
| kube-prometheus-stack | Prometheus, Grafana, Alertmanager | observability |
| loki-stack | Log aggregation with Promtail | observability |
| NGINX | Ingress controller | ingress |
| Solana RPC | Mainnet RPC node | blockchain |

## Prerequisites

- Kubernetes cluster (1.27+)
- Helm 3.x
- kubectl configured
- Storage classes available (standard + nvme-fast for Solana)

## Quick Start

### 1. Create namespaces

```bash
kubectl apply -f namespaces.yaml
```

### 2. Deploy observability stack

```bash
# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n observability \
  -f observability/kube-prometheus-stack/values.yaml

# Install Loki stack
helm install loki grafana/loki-stack \
  -n observability \
  -f observability/loki-stack/values.yaml
```

### 3. Deploy ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress \
  -f ingress/nginx/values.yaml
```

### 4. Deploy Solana RPC

```bash
# Create validator keypair secret (see secret.yaml.example)
solana-keygen new --outfile validator-keypair.json
kubectl create secret generic solana-validator-keypair \
  --from-file=validator-keypair.json \
  -n blockchain

# Deploy Solana RPC
kubectl apply -f blockchain/solana-rpc/
```

## Solana RPC Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 12 cores | 16+ cores |
| Memory | 256 GB | 512 GB |
| Storage | 2 TB NVMe | 4 TB NVMe |
| Network | 1 Gbps | 10 Gbps |

**Note:** Solana mainnet sync takes several days with a snapshot, weeks without.

## Configuration

### Update domains

Replace `example.com` in:
- `observability/kube-prometheus-stack/values.yaml` (Grafana ingress)

### Update storage classes

Adjust `storageClassName` in values files to match your cluster:
- `standard` — general purpose
- `nvme-fast` — high-performance NVMe (required for Solana)

## Monitoring

### Grafana Dashboards

Access Grafana at your configured domain. Pre-configured datasources:
- Prometheus: `http://prometheus-kube-prometheus-prometheus:9090`
- Loki: `http://loki-stack:3100`

### Alerts

Solana-specific alerts included:
- `SolanaRPCDown` — RPC endpoint unreachable
- `SolanaHighSlotLag` — Node falling behind network
- `SolanaHighMemoryUsage` — Memory above 90%
- `SolanaStorageAlmostFull` — Less than 10% storage remaining

## Security

- Secrets are **not** committed — use `secret.yaml.example` as template
- Consider using sealed-secrets or external-secrets for production
- RPC is private by default — expose via ingress as needed
- Network policies not included — add based on your requirements

## License

MIT
