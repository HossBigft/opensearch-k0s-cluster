To generate k0sctl config

```
bash -c 'set -a; source .env; envsubst < k0sctl.yaml.template > k0sctl.yaml'
```

To generate kubectl context
```bash
mkdir -p ~/.kube/ &&  k0sctl kubeconfig> ~/.kube/config
```