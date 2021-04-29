# Configure Elasticsearch Secret Engine for KubeVault

### necessary ENV variables
- `export REGISTRY=sakibalamin`
- `export VAULT_ADDR="https://127.0.0.1:8200"`
- `export VAULT_SKIP_VERIFY=true`
- `echo "cy5KaXVSSXRjZUxRcnJYM2c0S0RZZDlFYmw=" | base64 -d` decode the base64 encoded root token
- `export VAULT_TOKEN=s.JiuRItceLQrrX3g4KDYd9Ebl`
  
### steps
- `make push install` - push to dockerhub, alternatively - `make push-to-kind`
- `kubectl get all -n kube-system` & see that kubevault-community is running
- `kubectl logs -f -n kube-system pod/kubevault-community-594ccd9949-p5z92` watch operator logs

- `watch -n 1 kubectl get all -n kube-system` watch all resources under the kube-system namespace
- `kubectl create ns demo` creates the demo namespace
- `kubectl apply -f config.yaml` && `kubectl apply -f vs-version.yaml`
- ` kubectl get secrets -n demo vault-keys -o yaml` - get root token & unseal keys
- `kubectl get ns kube-system -o=jsonpath='{.metadata.uid}'` - get your kind cluster ID
- get the community license for KubeDB & put it in a file `license.txt`
- `helm repo add appscode https://charts.appscode.com/stable/`
- `helm repo update`
- `helm search repo appscode/kubedb`

- ``` 
  helm install kubedb appscode/kubedb \
  --version v2021.04.16      \
  --namespace kube-system                       \
  --set-file global.license=/path/to/the/license.txt
  ```

- `kubectl get elasticsearchversions` - get the elasticsearch versions
- `kubectl apply -f https://github.com/kubedb/docs/raw/v2021.04.16/docs/guides/elasticsearch/quickstart/overview/yamls/elasticsearch.yaml` - create elasticsearch CR
- `kubectl port-forward -n demo svc/es-quickstart 9200` - port forward to local machine
- `kubectl get secret -n demo es-quickstart-elastic-cred -o jsonpath='{.data.username}' | base64 -d` - `Username`: `elastic`
- `kubectl get secret -n demo es-quickstart-elastic-cred -o jsonpath='{.data.password}' | base64 -d` - `Password`: `q6XreFWkWi$;BsQy`
- `$ curl -XGET -k -u '<username>:<password>' "https://localhost:9200/_cluster/health?pretty"` - Health Check

- `kubectl apply -f secret-engine.yaml`

- `vault policy list`
```
default
k8s.-.demo.es-quickstart
k8s.-.demo.vault-auth-method-controller
vault-policy-controller
root
```
- `vault policy read k8s.-.demo.es-quickstart`
```
path "database/config/*" {
        capabilities = ["create", "update", "read", "delete"]
}

path "database/roles/*" {
        capabilities = ["create", "update", "read", "delete"]
}

path "database/creds/*" {
        capabilities = ["create", "update", "read"]
}

path "/sys/leases/*" {
  capabilities = ["create","update"]
}
```

- `vault auth list`
```
approle-path/    approle       auth_approle_3f575544       n/a
kubernetes/      kubernetes    auth_kubernetes_ee061cd8    n/a
token/           token         auth_token_47fce597         token based credentials
```

- `vault list auth/kubernetes/role`
```
Key                                 Value
---                                 -----
bound_service_account_names         [vault]
bound_service_account_namespaces    [demo]
token_bound_cidrs                   []
token_explicit_max_ttl              0s
token_max_ttl                       24h
token_no_default_policy             false
token_num_uses                      0
token_period                        24h
token_policies                      [default vault-policy-controller k8s.-.demo.es-quickstart]
token_ttl                           24h
token_type                          default

```

### resources:
- [Elasticsearch Database Secrets Engine](https://www.vaultproject.io/docs/secrets/databases/elasticdb)
- [Elasticsearch Quickstart using KubeDB](https://kubedb.com/docs/v2021.04.16/guides/elasticsearch/quickstart/overview/) 
