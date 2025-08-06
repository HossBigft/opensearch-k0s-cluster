To generate config

```
bash -c 'set -a; source .env; envsubst < k0sctl.yaml.template > k0sctl.yaml'
```