# Local Kubernetes Lab on Windows WSL

This Vagrant file will provision a working kubernetes cluster consisting of 1 controlplane node and 2 worker nodes. Vagrant file will run ansible to install and configure kubernetes on all nodes using kubeadm.

## Requirements

1. WSL (tested on Ubuntu)
2. VirtualBox installed on Windows Host
3. Vagrant installed on WSL
4. Ansible installed on WSL

## Steps

1. Create this file in wsl to fix error *"private key not secure"*. Resolves permissions issue since files are on windows host and not WSL

*/etc/wsl.conf*
```
[automount]  
options = "metadata,umask=22,fmask=11"
```

2. Append snippet into wsl .bashrc or .zshrc

```
export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"  
export VAGRANT_WSL_WINDOWS_ACCESS_USER_HOME_PATH="/mnt/c/Users/my_username/"  
export PATH="$PATH:/mnt/d/Program Files/Oracle/VirtualBox"
```

3. On windows, go to Control Panel\All Control Panel Items\Windows Defender Firewall\Allowed apps. Check all VirtualBox items (e.g VirtualBox Headless Frontend, VirtualBox Virtual Machine, etc.)

4. On WSL, run `vagrant plugin install virtualbox_WSL2`

5. Run Vagrant, while on repository root folder type `vagrant up`

6. If ansible fails, type `vagrant up --provision` to run only ansible

7. Type `vagrant destroy` to delete and clean up nodes

## Ansible
Ansible inventory is auto generated in the .vagrant folder
```
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory {{ playbook }}
```

## Keycloak Troubleshooting
Patch the configmap to add ENV KEYCLOAK_PROXY
1. Get the configmap deployed for keycloak
```
$ kubectl get configmap -n <namespace>                        
NAME                DATA   AGE
keycloak-env-vars   15     xx
```
2. Ensure you are doing it in the namespace where keycloak is deployed
```
kubectl patch configmap keycloak-env-vars -n <namespace> --type merge --patch '{"data":{"KEYCLOAK_PROXY":"edge"}}'
```

3. Get the pod using configmap and restart. You can delete the pod or scale the stateful set
```
$ kubectl get pods -n <namespace>
NAME                    READY   STATUS    RESTARTS   AGE
keycloak-0              1/1     Running   0          xx
keycloak-postgresql-0   1/1     Running   0          xx

$ kubectl delete pod keycloak-0 -n <namespace>
```

## oidc-login
Install krew
```
kubectl krew install oidc-login
kubectl oidc-login setup \
--oidc-issuer-url=https://{{ keycloak_domain }}/realms/master \
--oidc-client-id={{ client_id }} \
--oidc-client-secret={{ client_secret }} \
--insecure-skip-tls-verify

```

## Troubleshoot oidc-login
Clear cache
```
rm -rf ~/.kube/cache
rm -rf ~/.kube/http-cache
```

## Ingress and Cert-Manager
Make sure to indicate ingress class name and cert-manager issuer in ingress resource and its annotations
```
ingressClassName: nginx
annotations: 
    # add an annotation indicating the issuer to use.
    cert-manager.io/cluster-issuer: ca-issuer
enabled: true
hostname: "{{ keycloak_domain }}"
tls: true
extraTls: # < placing a host in the TLS config will determine what ends up in the cert's subjectAltNames. 
# Certmanager will generate secret with name keycloak.localdev.me-tls as initial TLS
    - hosts:
        - "{{ keycloak_domain }}"
    secretName: keycloak.cluster-tls
```

## Rancher Local Path Provisioner
Create pvc using storage class local-path

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 128Mi
```

## Run Ansible Manually
```
ansible-playbook  ansible/site.yaml --list-tags
ansible-playbook example.yml --tags "configuration,packages" --list-tasks
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory ansible/site.yaml
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory ansible/site.yaml --tags "configuration,packages"
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory ansible/site.yaml --tags "all,never"
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory ansible/site.yaml --skip-tags "packages"
```

## Prometheus and Grafana
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack \
  --create-namespace \
  --namespace kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack
kubectl -n kube-prometheus-stack get pods
```
Running a Prometheus Query
```
kubectl port-forward -n kube-prometheus-stack svc/kube-prometheus-stack-prometheus 9090:9090
```
Using Grafana Dashboards
```
kubectl port-forward -n svc/kube-prometheus-stack-grafana 8080:80
```

## References:
https://stackoverflow.com/questions/73883728/keycloak-admin-console-loading-indefinitely

https://medium.com/@charled.breteche/kind-keycloak-securing-kubernetes-api-server-with-oidc-371c5faef902

https://github.com/int128/kubelogin/issues/29

https://andrewtarry.com/posts/custom-dns-in-kubernetes/

https://github.com/rancher/local-path-provisioner

https://spacelift.io/blog/prometheus-kubernetes