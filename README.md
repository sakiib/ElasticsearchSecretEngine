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

- `vault secrets list` - we have our custom-database-path enabled
```
Path                     Type         Accessor              Description
----                     ----         --------              -----------
cubbyhole/               cubbyhole    cubbyhole_3f1ae146    per-token private secret storage
custom-database-path/    database     database_ee52628a     n/a
identity/                identity     identity_a5757022     identity store
sys/                     system       system_542c9d68       system endpoints used for control, policy and debugging
```

- `vault list custom-database-path/roles` - see the roles created after applying ElasticsearchRole CRD yaml

```
Keys
----
k8s.-.demo.es-quickstart

``` 

- ` vault read custom-database-path/roles/k8s.-.demo.es-quickstart`
```
Key                      Value
---                      -----
creation_statements      [{ "db": "admin", "roles": [{ "role": "readWrite" }, {"role": "read", "db": "foo"}] }]
db_name                  k8s.-.demo.es-quickstart
default_ttl              1h
max_ttl                  24h
renew_statements         []
revocation_statements    []
rollback_statements      []

```

- upto role creation
```
NAME                  READY   STATUS    RESTARTS   AGE
pod/es-quickstart-0   1/1     Running   1          29h
pod/es-quickstart-1   1/1     Running   1          29h
pod/es-quickstart-2   1/1     Running   1          29h
pod/vault-0           3/3     Running   3          29h

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
service/es-quickstart          ClusterIP   10.99.60.105     <none>        9200/TCP                        29h
service/es-quickstart-master   ClusterIP   None             <none>        9300/TCP                        29h
service/es-quickstart-pods     ClusterIP   None             <none>        9200/TCP                        29h
service/vault                  NodePort    10.104.214.202   <none>        8200:31809/TCP,8201:32153/TCP   29h
service/vault-internal         ClusterIP   None             <none>        8200/TCP,8201/TCP               29h
service/vault-stats            ClusterIP   10.104.105.19    <none>        56790/TCP                       29h

NAME                             READY   AGE
statefulset.apps/es-quickstart   3/3     29h
statefulset.apps/vault           1/1     29h

NAME                                               TYPE                       VERSION   AGE
appbinding.appcatalog.appscode.com/es-quickstart   kubedb.com/elasticsearch   7.9.1     29h
appbinding.appcatalog.appscode.com/vault                                                29h

NAME                                              STATUS    AGE
secretengine.engine.kubevault.com/es-quickstart   Success   3h13m

NAME                                                   STATUS    AGE
elasticsearchrole.engine.kubevault.com/es-quickstart   Success   84m

NAME                              REPLICAS   VERSION   STATUS    AGE
vaultserver.kubevault.com/vault   1          1.7.0     Running   29h

NAME                                                                   STATUS    AGE
vaultpolicybinding.policy.kubevault.com/vault-auth-method-controller   Success   29h

NAME                                                            STATUS    AGE
vaultpolicy.policy.kubevault.com/vault-auth-method-controller   Success   29h

NAME                                     VERSION          STATUS   AGE
elasticsearch.kubedb.com/es-quickstart   xpack-7.9.1-v1   Ready    29h

```

### resources:
- [Elasticsearch Database Secrets Engine](https://www.vaultproject.io/docs/secrets/databases/elasticdb)
- [Elasticsearch Quickstart using KubeDB](https://kubedb.com/docs/v2021.04.16/guides/elasticsearch/quickstart/overview/) 
