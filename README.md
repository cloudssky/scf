# Cloud Foundry on Kubernetes Deployment Guide

This is a step by step deployment guide to setup Cloud Foundry on top of EKS or TK8 made Kubernetes Clusters.

If you don't have a k8s cluster running, we made the life easier for you:

tk8eks: installs eks in 10 minutes:

https://github.com/kubernauts/tk8eks

tk8: installs upstream k8s on AWS, Bare-Metal or OpenStack in 8 minutes:

https://github.com/kubernauts/tk8

## Steps:

1. Configure kubectl to use EKS cluster

2. Get the Public IP & private ( IP used the private DNS name)  for any EKS node except master

3. Create a DNS record on the test Domain

4. Update A record *.<your-test-domain>  (eg *.kubernauts.org) pointing to the node’s Public IP)

5. Create scf-config-values.yaml ( https://github.com/cloudssky/scf/blob/master/scf-config-values.yaml)

** note that Domain name is <public_ip_of_eks_node> from step 2

* external_ips: ["10.0.1.103"] is Private IP of EKS Node from step 2

Create persistent storage class using (adapt the zone):

`kubectl create -f storage-class.yaml`

Source file: https://github.com/cloudssky/scf/blob/master/storage-class.yaml

Test the Persistent Storage Class created by :

`kubectl create -f test-storage-class.yaml`

Source file: https://github.com/cloudssky/scf/blob/master/test-storage-class.yaml

Check if it’s successfully created by:

`kubectl get pv,pvc`

Delete the PVC with:

`kubectl  delete -f test-storage-class.yaml`

Create the following security rules on the Security Group for the EKS node:


```bash
HTTP TCP	80	0.0.0.0/0	CAP HTTP
HTTPS	TCP	443	0.0.0.0/0	CAP HTTPS
Custom TCP Rule	TCP	2793	0.0.0.0/0	CAP UAA
Custom TCP Rule	TCP	2222	0.0.0.0/0	CAP SSH
Custom TCP Rule	TCP	4443	0.0.0.0/0	CAP WSS
Custom TCP Rule TCP	20000 - 20009	0.0.0.0/0	CAP TCP Routing
```

### optional

Increase the EBS volume size of all nodes from 20GB to 60GB:
Enable ssh on the nodes by opening SSH port 22 on the Security Group for nodes

SSH in to the nodes then:

`sudo growpart /dev/nvme0n1 1 && sudo xfs_growfs -d /`

This steps can be different depending on the AMI or file system refer:

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html

Install Helm:


```bash
helm init
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

### Add suse repo and verify:

```bash
helm repo add suse https://kubernetes-charts.suse.com/
helm repo list
```

### Create Name Spaces for SCF and UAA:

```bash
kubectl create ns scf
Kubectl create ns uaa
```

### Install UAA with helm:

```bash
helm install suse/uaa  --name susecf-uaa --namespace uaa --values scf-config-values.yaml
```

Check if all the pods are ready:

```bash
kubectl -n uaa get po -w
```

Wait until all pods are ready

Output like:

```bash
NAME                        READY STATUS RESTARTS   AGE
 
mysql-0                     1/1 Running 0     3m
 
mysql-proxy-0               1/1 Running 0     3m
 
secret-generation-1-lmrhl   0/1 Completed 0     3m
 
uaa-0                       0/1 Running 0     3m
```

### Install scf when all pods are ready:

```bash
SECRET=$(kubectl get pods --namespace uaa -o jsonpath='{.items[*].spec.containers[?(.name=="uaa")].env[(.name=="INTERNAL_CA_CERT")].valueFrom.secretKeyRef.name}')
CA_CERT="$(kubectl get secret $SECRET --namespace uaa -o jsonpath="{.data['internal-ca-cert']}" | base64 --decode -)"
 helm install suse/cf --name susecf-scf --namespace scf --values scf-config-values.yaml --set "secrets.UAA_CA_CERT=${CA_CERT}"
 ```

Wait until all pods are ready:

```bash
kubectl get pods --namespace scf -w
```

### Test with cf cli:

Install CF CLI for your OS

```bash
cf api --skip-ssl-validation https://api.35.162.169.236.kubernauts.org
cf login
```

e-mail: admin

Password: password

Install Stratos Web UI:

```bash
kubectl create namespace stratos
```

Check if you have Helm charts for Startos (suse/console):

```bash
helm search suse
```

### Install Stratos using helm

```bash
helm install suse/console  --name susecf-console --namespace stratos --values scf-config-values.yaml  --set storageClass=gp2
```

*note use the persistent storage class name created with step 5


Related links

https://www.suse.com/documentation/cloud-application-platform-1/book_cap_deployment/data/cha_cap_eks.html

https://github.com/SUSE/scf/wiki/Deployment-on-Amazon-EKS
