# Supabase Kubernetes

This repository contains the charts to deploy a [Supabase](https://github.com/supabase/supabase) instance inside a Kubernetes cluster using Helm 3.

For any information regarding Supabase itself you can refer to the [official documentation](https://supabase.io/docs).

## What's Supabase ?

Supabase is an open source Firebase alternative. We're building the features of Firebase using enterprise-grade open source tools.

## How to use ?

You can find the documentation inside the [chart directory](./charts/supabase/README.md)

## AKS Deployment (talentsphere-test)

### Prerequisites

- Azure CLI (`az`)
- kubectl
- Helm 3

### Quick Start

1. **Login to Azure and connect to AKS**

   ```bash
   az login
   az aks get-credentials --resource-group <YOUR_RESOURCE_GROUP> --name <YOUR_AKS_CLUSTER_NAME>
   ```

2. **Create the namespace**

   ```bash
   kubectl create namespace talentsphere-test
   ```

3. **Deploy Supabase**

   ```bash
   helm install supabase ./charts/supabase -f values-talentsphere-test.yaml -n talentsphere-test
   ```

4. **Verify deployment**
   ```bash
   kubectl get pods -n talentsphere-test
   ```

### Access Supabase

**Port forward for local access:**

```bash
# API Gateway (Kong)
kubectl port-forward svc/supabase-supabase-kong -n talentsphere-test 8000:8000

# Studio Dashboard
kubectl port-forward svc/supabase-supabase-studio -n talentsphere-test 3000:3000
```

### Components Deployed

| Component        | Service                     | Port |
| ---------------- | --------------------------- | ---- |
| PostgreSQL       | supabase-supabase-db        | 5432 |
| Studio           | supabase-supabase-studio    | 3000 |
| Auth (GoTrue)    | supabase-supabase-auth      | 9999 |
| REST (PostgREST) | supabase-supabase-rest      | 3000 |
| Realtime         | supabase-supabase-realtime  | 4000 |
| Meta             | supabase-supabase-meta      | 8080 |
| Storage          | supabase-supabase-storage   | 5000 |
| imgproxy         | supabase-supabase-imgproxy  | 5001 |
| Kong             | supabase-supabase-kong      | 8000 |
| Analytics        | supabase-supabase-analytics | 4000 |
| Vector           | supabase-supabase-vector    | 9001 |
| Functions        | supabase-supabase-functions | 9000 |

### Configuration Notes

- **Node Selector**: Pods scheduled on `userpool` nodes (`kubernetes.azure.com/agentpool: userpool`)
- **Storage**: Uses Azure managed disks (`managed-csi` storage class)
- **Ingress**: Configured for `talentsphere-test.example.com` (update for your domain)

### Upgrade

```bash
helm upgrade supabase ./charts/supabase -f values-talentsphere-test.yaml -n talentsphere-test
```

### Uninstall

```bash
helm uninstall supabase -n talentsphere-test
kubectl delete namespace talentsphere-test
```

## Weaviate Deployment

Weaviate is deployed as a separate Helm release for vector search capabilities.

### Install Weaviate

```bash
helm repo add weaviate https://weaviate.github.io/weaviate-helm
helm repo update
helm install weaviate weaviate/weaviate -n talentsphere-test -f weaviate-values-talentsphere-test.yaml
```

### Important Configuration Notes

For Weaviate v1.25+ running on Kubernetes, the RAFT consensus mechanism requires proper DNS resolution. The following environment variables are required in your values file:

```yaml
env:
  RAFT_ENABLE_FQDN_RESOLVER: "true"
  RAFT_FQDN_RESOLVER_TLD: weaviate-headless.<namespace>.svc.cluster.local
```

**Do not** set a custom `CLUSTER_HOSTNAME` - the Helm chart automatically configures this based on the pod name.

### Troubleshooting Weaviate Bootstrap Failures

If Weaviate pods fail with `unable to resolve any node address to join` errors:

1. Ensure `RAFT_ENABLE_FQDN_RESOLVER` is set to `"true"`
2. Verify `RAFT_FQDN_RESOLVER_TLD` matches your headless service FQDN
3. If the pod keeps restarting, clear corrupted RAFT state:
   ```bash
   kubectl exec -n <namespace> weaviate-0 -- rm -rf /var/lib/weaviate/raft
   kubectl delete pod weaviate-0 -n <namespace>
   ```

### Weaviate Services

| Component              | Service                        | Port  |
| ---------------------- | ------------------------------ | ----- |
| Weaviate               | weaviate                       | 80    |
| Weaviate gRPC          | weaviate-grpc                  | 50051 |
| Transformers Inference | weaviate-text2vec-transformers | 8080  |

### Upgrade Weaviate

```bash
helm upgrade weaviate weaviate/weaviate -n talentsphere-test -f weaviate-values-talentsphere-test.yaml
```

### Uninstall Weaviate

```bash
helm uninstall weaviate -n talentsphere-test
kubectl delete pvc weaviate-data-weaviate-0 -n talentsphere-test
```

# Roadmap

- [ ] Multi-node Support

## Support

This project is supported by the community and not officially supported by Supabase. Please do not create any issues on the official Supabase repositories if you face any problems using this project, but rather open an issue on this repository.

## Contributing

You can contribute to this project by forking this repository and opening a pull request.

When you're ready to publish your chart on the `main` branch, you'll have to execute `sh build.sh` to package the charts and generate the Helm manifest.

## License

[Apache 2.0 License.](./LICENSE)
