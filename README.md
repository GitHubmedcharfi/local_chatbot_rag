# Local AI Platform - Approach 1: k3s with raw manifests

CPU-only local chatbot and RAG on a single VM, deployed with plain Kubernetes YAML on k3s.
LibreChat is the chat UI. No custom frontend or backend.
This is the baseline for the Helm (approach 2) and Juju (approach 3) branches.

## Architecture

Namespace: `local-ai`

    User -> Traefik -> librechat (:3080) -> llm-chat (:8080) -> answer
    librechat -> mongodb (:27017)           users and conversations
    librechat -> rag-api (:8000) -> llm-embed (:8080) -> vectors -> postgresql + pgvector (:5432)

| Component | Kind | Image / notes |
|---|---|---|
| postgresql | StatefulSet | `pgvector/pgvector:pg16`, user `raguser`, DB `ragdb`, 30Gi hostPath PV (`storageClassName: manual`) |
| mongodb | StatefulSet | `mongo:7.0` (AVX present; use `mongo:4.4` if `grep -c avx /proc/cpuinfo` is 0), 10Gi PV |
| llm-chat | Deployment | llama.cpp server, quantized GGUF chat model, init container downloads it to a PVC |
| llm-embed | Deployment | llama.cpp server in embedding mode, small embedding model |
| rag-api | Deployment | LibreChat RAG API, `VECTOR_DB_TYPE=pgvector`, embeddings via `http://llm-embed:8080/v1` |
| librechat | Deployment | LibreChat, uploads PVC, `librechat.yaml` custom endpoint to `http://llm-chat:8080/v1` |
| Ingress | Traefik (bundled with k3s) | host rule and TLS |
| Monitoring | Prometheus + Grafana | CPU, RAM, latency, DB dashboards |

Excluded: Ollama, vLLM, GPU components, Meilisearch.

## Requirements

- VM with k3s (keep the bundled Traefik), `kubectl`
- No GPU. Measure first: `nproc`, `free -h`, `df -h /`
- Size the chat model to the VM: 1B to 4B parameters, Q4-family GGUF. Set llama.cpp
  threads (`-t`) close to the cores available. Keep models on a PVC so restarts do not
  download them again.

## Repository layout

    k8s/
      namespace.yaml
      postgresql/    secret, configmap, pv, pvc, statefulset, service
      mongodb/       secret, pv, pvc, statefulset, service
      llm-chat/
      llm-embed/
      rag-api/       configmap, deployment, service
      librechat/     secret, configmap, pvc, deployment, service
      traefik/
    monitoring/      prometheus/, grafana/dashboards/
    docs/            architecture.md, benchmarks.md

`secret.yaml` files are in `.gitignore`. Only `secret.example.yaml` files are committed.
Create the real `secret.yaml` files from the examples and set your own values before applying.

## Install

Apply in this order:

    kubectl apply -f k8s/namespace.yaml
    kubectl apply -f k8s/postgresql/
    kubectl apply -f k8s/mongodb/
    kubectl apply -f k8s/llm-chat/
    kubectl apply -f k8s/llm-embed/
    kubectl apply -f k8s/rag-api/
    kubectl apply -f k8s/librechat/
    kubectl apply -f k8s/traefik/

Check:

    kubectl -n local-ai get pods,svc,pvc,ingress
    kubectl -n local-ai exec -it postgresql-0 -- sh -c 'psql -U raguser -d ragdb -c "\dx"'

`\dx` should list `vector`. Reach LibreChat through the Ingress, or with
`kubectl -n local-ai port-forward svc/librechat 3080:3080`. Set `DOMAIN_CLIENT` and
`DOMAIN_SERVER` to the real hostname.

## Uninstall

    kubectl delete namespace local-ai
    kubectl delete pv --all   # only the PVs created for this project

hostPath PVs are not removed with the namespace. Delete them explicitly if you want a clean disk.

## Verification status

| Step | What | Status |
|---|---|---|
| 1.1 | k3s, Traefik, namespace | CHECK |
| 1.2 | PostgreSQL + pgvector (`\dx` shows `vector`) | CHECK |
| 1.3 | MongoDB (pod Ready, port 27017) | CHECK |
| 1.4 | llm-chat (`/v1/models`, `/v1/chat/completions`) | CHECK |
| 1.5 | llm-embed (`/v1/embeddings` returns a vector) | CHECK |
| 1.6 | rag-api (Ready, no DB errors) | CHECK |
| 1.7 | LibreChat (UI reachable, data persists after restart) | CHECK |
| 1.8 | Chat answered by the local model | CHECK |
| 1.9 | Ingress with TLS, iframe embedding test | CHECK |
| 1.10 | RAG test: grounded answer, rows in pgvector | CHECK |
|
