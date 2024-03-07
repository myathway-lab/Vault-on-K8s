## Install Vault on Kubernetes Cluster

### Summary

In this lab, we will deploy vault cluster on Kubernetes using Helm.  <br>
We will test the AWS secret engine to rotate the access keys.  <br>
We will test vault role to manage IAM access.  <br>


1) Add Hashicorp repo using helm chart.
2) Create the helm value file to customize the default vaules.
3) Install vault using helm.
4) Unseal the first vault node. 
5) Join other nodes to the vault cluster. 
6) Unseal another nodes.
7) Access vault UI.
8) Enable AWS engine.
9) Manage the AWS secret engine & Rotate IAM access key
10) Create a role inside Vault. 
11) Use role to create AWS IAM access. 

<br>


**Requirements** 

- Make sure K8s cluster ready.
- Install Helm.

```yaml
vagrant@kindcluster-box:~/kind-demo/scripts$ kubectl get no
NAME                STATUS   ROLES           AGE   VERSION
127-control-plane   Ready    control-plane   10m   v1.27.3
127-worker          Ready    <none>          10m   v1.27.3
127-worker2         Ready    <none>          10m   v1.27.3
127-worker3         Ready    <none>          10m   v1.27.3
```

<br>

**1) Add Hashicorp repo using helm chart.**

```yaml
vagrant@kindcluster-box:~/kind-demo/scripts$ helm repo add hashicorp [https://helm.releases.hashicorp.com](https://helm.releases.hashicorp.com/)
"hashicorp" has been added to your repositories
vagrant@kindcluster-box:~/kind-demo/scripts$ helm repo list
NAME            URL
hashicorp       [https://helm.releases.hashicorp.com](https://helm.releases.hashicorp.com/)

vagrant@kindcluster-box:~/kind-demo/scripts$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "hashicorp" chart repository
Update Complete. ⎈Happy Helming!⎈

vagrant@kindcluster-box:~/kind-demo/scripts$ helm search repo hashicorp
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
hashicorp/consul                        1.3.2           1.17.2          Official HashiCorp Consul Chart
hashicorp/terraform                     1.1.2                           Install and configure Terraform Cloud Operator ...
hashicorp/terraform-cloud-operator      2.2.0           2.2.0           Official Helm chart for HashiCorp Terraform Clo...
hashicorp/terraform-enterprise          1.1.1           1.16.0          Official HashiCorp Terraform-Enterprise Chart
hashicorp/vault                         0.27.0          1.15.2          Official HashiCorp Vault Chart
hashicorp/vault-secrets-operator        0.4.3           0.4.3           Official Vault Secrets Operator Chart
hashicorp/waypoint                      0.1.21          0.11.3          Official Helm Chart for HashiCorp Waypoint

vagrant@kindcluster-box:~/kind-demo/scripts$ helm search repo hashicorp/vault  --versions
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
hashicorp/vault                         0.27.0          1.15.2          Official HashiCorp Vault Chart
hashicorp/vault                         0.26.1          1.15.1          Official HashiCorp Vault Chart
hashicorp/vault                         0.26.0          1.15.1          Official HashiCorp Vault Chart
hashicorp/vault                         0.25.0          1.14.0          Official HashiCorp Vault Chart
hashicorp/vault                         0.24.1          1.13.1          Official HashiCorp Vault Chart
hashicorp/vault                         0.24.0          1.13.1          Official HashiCorp Vault Chart
hashicorp/vault                         0.23.0          1.12.1          Official HashiCorp Vault Chart
hashicorp/vault                         0.22.1          1.12.0          Official HashiCorp Vault Chart
hashicorp/vault                         0.22.0          1.11.3          Official HashiCorp Vault Chart
hashicorp/vault                         0.21.0          1.11.2          Official HashiCorp Vault Chart
hashicorp/vault                         0.20.1          1.10.3          Official HashiCorp Vault Chart
hashicorp/vault                         0.20.0          1.10.3          Official HashiCorp Vault Chart
hashicorp/vault                         0.19.0          1.9.2           Official HashiCorp Vault Chart
hashicorp/vault                         0.18.0          1.9.0           Official HashiCorp Vault Chart
hashicorp/vault                         0.17.1          1.8.4           Official HashiCorp Vault Chart
hashicorp/vault                         0.17.0          1.8.4           Official HashiCorp Vault Chart
hashicorp/vault                         0.16.1          1.8.3           Official HashiCorp Vault Chart
hashicorp/vault                         0.16.0          1.8.2           Official HashiCorp Vault Chart
hashicorp/vault                         0.15.0          1.8.1           Official HashiCorp Vault Chart
hashicorp/vault                         0.14.0          1.8.0           Official HashiCorp Vault Chart
hashicorp/vault                         0.13.0          1.7.3           Official HashiCorp Vault Chart
hashicorp/vault                         0.12.0          1.7.2           Official HashiCorp Vault Chart
hashicorp/vault                         0.11.0          1.7.0           Official HashiCorp Vault Chart
hashicorp/vault                         0.10.0          1.7.0           Official HashiCorp Vault Chart
hashicorp/vault                         0.9.1           1.6.2           Official HashiCorp Vault Chart
hashicorp/vault                         0.9.0           1.6.1           Official HashiCorp Vault Chart
hashicorp/vault                         0.8.0           1.5.4           Official HashiCorp Vault Chart
hashicorp/vault                         0.7.0           1.5.2           Official HashiCorp Vault Chart
hashicorp/vault                         0.6.0           1.4.2           Official HashiCorp Vault Chart
hashicorp/vault                         0.5.0                           Install and configure Vault on Kubernetes.
hashicorp/vault                         0.4.0                           Install and configure Vault on Kubernetes.
hashicorp/vault-secrets-operator        0.4.3           0.4.3           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.4.2           0.4.2           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.4.1           0.4.1           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.4.0           0.4.0           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.3.4           0.3.4           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.3.3           0.3.3           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.3.2           0.3.2           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.3.1           0.3.1           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.2.0           0.2.0           Official Vault Secrets Operator Chart
hashicorp/vault-secrets-operator        0.1.0           0.1.0           Official Vault Secrets Operator Chart
```

<br>

**2) Create the helm value file to customize the default vaules.**

- helm-vault-raft-values.yaml
    
    ```yaml
    server:
      affinity: ""
      ha:
        enabled: true
        raft:         #for storage
          enabled: true  
    
    ui: 
      enabled: true   #default:false # replicas:3
      serviceType: "LoadBalancer"
    
    injector:         #app injected with sidecar engine
      enabled: true
    ```
    
<br>

**3) Install vault using helm.**

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ helm install vault02 hashicorp/vault --values helm-vault-raft-values.yaml -n vault
```

- Pods are not up & running yet because they are in sealed state.


```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl get po -n vault
NAME                                      READY   STATUS    RESTARTS   AGE
vault01-0                                 0/1     Running   0          96s
vault01-1                                 0/1     Running   0          95s
vault01-2                                 0/1     Running   0          95s
vault01-agent-injector-685fb4d478-6mxwd   1/1     Running   0          96s
```

- Verify vault status.

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-0 -n vault -- sh
/ $ vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.15.2
Build Date         2023-11-06T11:33:28Z
Storage Type       raft
HA Enabled         true
```

<br>

**4) Unseal the first vault node.**

- To unseal the node, we first need initialize the vault operator.

```yaml
kubectl exec -it vault01-0 -n vault -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```

- cluster-keys.json
    
    ```yaml
    {
      "unseal_keys_b64": [
        "eZdvxjEb6C7vSvHfDVx/EFTtBXsISD2/vWqH7WJlWz0="
      ],
      "unseal_keys_hex": [
        "79976fc6311be82eef4af1df0d5c7f1054ed057b08483dbfbd6a87ed62655b3d"
      ],
      "unseal_shares": 1,
      "unseal_threshold": 1,
      "recovery_keys_b64": [],
      "recovery_keys_hex": [],
      "recovery_keys_shares": 0,
      "recovery_keys_threshold": 0,
      "root_token": "hvs.FK6uMX6XDMBXhniOEMJRRgC0"
    }
    ```
    

- Then export unseal_keys_b64 & use it to unseal vault01-0 node.

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ export VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" cluster-keys.json)
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ echo $VAULT_UNSEAL_KEY
eZdvxjEb6C7vSvHfDVx/EFTtBXsISD2/vWqH7WJlWz0=
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-0 -n vault -- vault operator unseal $VAU
LT_UNSEAL_KEY
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.15.2
Build Date              2023-11-06T11:33:28Z
Storage Type            raft
Cluster Name            vault-cluster-beb70f2e
Cluster ID              23de244f-cfad-31ca-9984-d1de8b5ca89d
HA Enabled              true
HA Cluster              https://vault01-0.vault01-internal:8201
HA Mode                 active
Active Since            2024-02-11T10:00:47.234675373Z
Raft Committed Index    53
Raft Applied Index      53
```

<br>

**5) Join other nodes to the vault cluster.**

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-1 -n vault -- vault operator raft join http://vault01-0.vault01-internal:8200
Key       Value
---       -----
Joined    true
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-2 -n vault -- vault operator raft join http://vault01-0.vault01-internal:8200
Key       Value
---       -----
Joined    true
```

- We cannot list vault peer list for now because we haven't authenticated with vault using root token. 

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-2 -n vault -- vault operator raft list-peers
Error reading the raft cluster configuration: Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/storage/raft/configuration
Code: 503. Errors:

* Vault is sealed
command terminated with exit code 2
```

<br>

**6) Unseal another nodes.**

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-1 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.15.2
Build Date         2023-11-06T11:33:28Z
Storage Type       raft
HA Enabled         true
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-2 -n vault -- vault operator unseal $VAU
LT_UNSEAL_KEY
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.15.2
Build Date         2023-11-06T11:33:28Z
Storage Type       raft
HA Enabled         true
```

- Now all the vault nodes are fully up & running. 

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl get po -n vault
NAME                                      READY   STATUS    RESTARTS   AGE
vault01-0                                 1/1     Running   0          7m25s   
vault01-1                                 1/1     Running   0          7m24s   
vault01-2                                 1/1     Running   0          7m24s   
vault01-agent-injector-685fb4d478-6mxwd   1/1     Running   0          7m25s   
```

**Note - vault uses stateful set because it will use RAFT integrated storage which is used to store secrets.**


<br>

**7) Login to the vault UI.**

- Port forward to access from our laptop.

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl port-forward svc/vault01 -n vault 8200:8200 --address 0.0.0.0
Forwarding from 0.0.0.0:8200 -> 8200
Handling connection for 8200
Handling connection for 8200
Handling connection for 8200
Handling connection for 8200
```

![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/7c0c999c-33ab-47a6-a957-ef614af99a6a)

![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/e403e88b-303e-4b1c-a342-c5897ac5d711)

- Login to vault using root token.
  
```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-0 -n vault -- vault login
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.FK6uMX6XxxxxxxxxxxxJRRgC0
token_accessor       W2GhOwgxxxxxxxxxxBWxm35rw
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

<br>

**8) Enable AWS engine.**

- We will enable AWS engine to store AWS access keys inside vault.

```yaml
vagrant@kindcluster-box:~/kind-demo/k8s-vault-demo01$ kubectl exec -it vault01-0 -n vault -- sh
/ $ export VAULT_ADDR="http://127.0.0.1:8200"
/ $ echo $VAULT_ADDR
http://127.0.0.1:8200
/ $ export AWS_ACCESS_KEY_ID=AKIAQ3xxxxxxxxxxEKWY
/ $ export AWS_SECRET_ACCESS_KEY=Y+as6s+1xxxxxxxxxxxxxxxxxxxrKvT6

/ $ vault secrets enable -path=mt-aws-master aws
Success! Enabled the aws secrets engine at: mt-aws-master/
/ $ vault secrets list
Path              Type         Accessor              Description
----              ----         --------              -----------
cubbyhole/        cubbyhole    cubbyhole_095d8583    per-token private secret storage
identity/         identity     identity_628ca529     identity store
mt-aws-master/    aws          aws_fd32ce87          n/a
sys/              system       system_37476832       system endpoints used for control, policy and debugging
/ $
```

- We will tune default-lease-ttl to 10min.

```yaml
/ $ vault secrets tune -default-lease-ttl=10m /mt-aws-master
Success! Tuned the secrets engine at: mt-aws-master/
/ $
```

![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/60166a1f-25a5-4d67-83ba-1abafad6b9b9)

<br>

**9) Manage the AWS secret engine & Rotate IAM access key.**

- below is my current AWS IAM access key.

![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/12d41fb8-8cd4-4aeb-97d4-7cb69ed007fd)

- Now let's write those access keys into Vault secret & rotate using vault.

```yaml
/ $ vault write mt-aws-master/config/root access_key=$AWS_ACCESS_KEY_ID secret_key=$AWS_SECRET_ACCESS_KEY region=ap-southeast-1
Success! Data written to: mt-aws-master/config/root

/ $ vault write -f mt-aws-master/config/rotate-root
Key           Value
---           -----
access_key    AKIAQ3xxxxxxxxxxEKWY
/ $
```

- Verify the access key is rotated the new key “AKIAQxxxxxxxxxxK2WX”.

![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/6eb10929-c1f2-4782-934b-ef404571aaa8)

<br>

**10) Create a role inside Vault.**

- We will create new vault AWS role inside Vault.

**By using this role, vault can create new IAM user. Policy can be defined as below. The secret lease duration follows the lease time of it secret engine.**



```yaml
$ vault write mt-aws-master/roles/aws-terraform-admin-role credential_type=iam_user \
> policy_document=-<<EOF
> {
>   "Version": "2012-10-17",
>   "Statement": [
>     {
>       "Effect": "Allow",
>       "Action":  "*",
>       "Resource": "*"
>     }
>     ]
> }
> EOF
Success! Data written to: mt-aws-master/roles/aws-terraform-admin-role
```

- Verify aws-terraform-admin-role is created.
  
![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/f69f5fcc-cce6-451c-98e2-347a6bdd2d03)

- For now, we have only two IAM accounts.
  
![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/731e8c5b-38fa-4858-b290-be0997a100d5)

<br>

**11) Use vault role to create AWS IAM access.**

```yaml
/ $ vault read mt-aws-master/creds/aws-terraform-admin-role 
Key                Value
---                -----
lease_id           mt-aws-master/creds/aws-terraform-admin-role/eTWDQcvB8NSFFZnpzVg90VXO
lease_duration     1h
lease_renewable    true
access_key         AKIAQxxxxxxxxxxxxxxOGVI
secret_key         pHhm7LGBJxxxxxxxxxxxxxxxxxleF5097WeLLUbE
security_token     <nil>
/ $
```

- Veriy the new IAM account was created.
  
![image](https://github.com/myathway-lab/Vault-on-K8s/assets/157335804/0b6424b4-1873-4767-bc3b-b988c4c4cc8c)
