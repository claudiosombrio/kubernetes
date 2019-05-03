# Kubernetes

### Instalar Kubernetes no Ubuntu 18.04

Adicionar repositório
```
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo “deb https://apt.kubernetes.io/ kubernetes-xenial main” | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```
Instalar versão específica
```
sudo apt-get install -qy kubectl=1.11.10-00
```
Verificar instalação
```
kubectl version
```
### Instalar Kops
Usado para criar cluster na AWS
```
wget https://github.com/kubernetes/kops/releases/download/1.11.0/kops-linux-amd64
chmod +x kops-linux-amd64
mv kops-linux-amd64 /usr/local/bin/kops
```
### Configurar
Adicionar variáveis de ambiente em `/etc/enviroments`

```
REGION=us-east-1
KOPS_STATE_STORE=s3://<bucket_name>
NAME=<cluster.com>
```
>NAME: Usado para o nome do cluster

Criar repositório para arquivos de configuração no S3
```
aws s3api create-bucket --bucket <bucket_name> --region ${REGION}
aws s3api put-bucket-versioning --bucket <bucket_name> --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket <bucket_name> --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### Criar cluster
Cria um cluster com 3 master em 2 zonas de disponibilidade.  
```
kops create cluster --zones us-east-1a,us-east-1d --topology private --networking calico --master-size t3.small --master-count 3 --node-size t3.small ${NAME}
```
Criar chave para acesso SSH nas master
```
kops create secret --name ${NAME} sshpublickey admin -i ~/.ssh/id_rsa.pub
```
Criar certificado SSL no AWS ACM `*.cluster.com`. Editar cluster para adicionar ARN do certificado SSL
```
...
spec:
  api:
    loadBalancer:
      sslCertificate: <arn:aws:acm...>
      type: Public
  authorization:
    rbac: {}
...
```

### Deploy do cluster
Visualizar os detalhes do cluster
```
kops update cluster ${NAME}
```
Confirmar deploy e criar efetivamente o cluster
```
kops update cluster ${NAME} --yes
```
Verificar situação do cluster
```
kops validate cluster
```
Visualizar configurações do cluster
```
kubectl config view
```

### Adicionar kubernetes dashboard - Ref: [Dashboard DOC](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
Criar usuário de acesso

>Criar arquivos com o conteúdo abaixo e rodar `kubectl apply -f <filename.yaml>`

Criação do usuário
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```
Vinculando usuário a função
* Role `cluster-admin` já deve ter sido criada pelo kops
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
Recuperar o token do usuário criado
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
Para acessar o dashboard na máquina local rodar `kubectl proxy` e acessar o endereço (http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)

