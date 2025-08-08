# Depends On

```mermaid
graph RL
	opensearch-certs_opensearch["opensearch-certs@opensearch"] --> cert-manager_cert-manager["cert-manager@cert-manager"]
	opensearch_opensearch["opensearch@opensearch"] --> opensearch-certs_opensearch["opensearch-certs@opensearch"]
	opensearch-dashboards_opensearch["opensearch-dashboards@opensearch"] --> opensearch_opensearch["opensearch@opensearch"]
```