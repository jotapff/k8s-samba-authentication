
# webhook service server 

## Instalar GO

```bash

wget https://dl.google.com/go/go1.14.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.14.linux-amd64.tar.gz
echo "PATH=$PATH:/usr/local/go/bin" >>~/.bashrc &&  . ~/.bashrc
```

```bash
#check
go version 
```

## Configurar o WebHook service 
```bash
git clone https://github.com/jotap1999/k8s-samba-authentication.git
cd k8s-samba-authentication

go get github.com/go-ldap/ldap
go get k8s.io/api/authentication/v1
```


Editar ficheiro  main.go  com base na configuação do LDAP/SAMBA 

```bash
nano main.go
#Line 18 - Se o servidor LDAP estiver configurado "over ssl/tls"
ldapURL = "ldaps://" + os.Args[1]

#Line 95
user :=  fmt.Sprintf("%s@KUBER.NET", username)  

#Line 104
"cn=Users,dc=kuber,dc=net"
```


## Compilar o código 
```bash
GOOS=linux GOARCH=amd64 go build main.go
```

## Gerar certificados
Criar self-signed certificate. É recomendado usar um certificado assinado por uma CA 
```bash
openssl req -x509 -newkey rsa:2048 -nodes \
    -subj "/CN=localhost" \
    -keyout key.pem \
    -out cert.pem
```
## Executar o WebHook service 
```bash
./main SERVER-LDAP  key.pem cert.pem  &>/var/log/k8s-samba-authentication.log &
```
## Test 
```bash
nano testldap.json
  {
    "apiVersion": "authentication.k8s.io/v1",
    "kind": "TokenReview",
    "spec": {
      "token": "user:userpassword"
    }
  }

curl -k -X POST -d @testldap.json https://127.0.0.1
# Se o status estiver vazio, o webhook não está a funcionar  
 "status": {
    "user": {}
  }

```
## Usar systemd para iniciar no BOOT

```bash 
nano /etc/systemd/system/webhook.service

[Unit]
Description=Samba AD Webhook Authentication Server
After=network.target

[Service]
Type=simple


ExecStart=/root/k8s-samba-authentication/main 127.0.0.1 /root/k8s-samba-authentication/key.pem /root/k8s-samba-authentication/cert.pem


RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash 
systemctl start webhook.service
systemctl enable webhook.service
```

# Kubernetes server

```bash
#Install kubeadm and Docker
curl -o- https://raw.githubusercontent.com/jotap1999/k8s-docker-Install-Script-Ubuntu/master/install.sh  | bash
```

## Criar ficheiro de configuração do Webhook Token
```bash
cat <<EOF > /root/webhook-config.yaml
apiVersion: v1
kind: Config
clusters:
  - name: authn
    cluster:
      server: https://X.X.X.X       #WebHook Server
      insecure-skip-tls-verify: true    #Se o certificado não estiver assinado
users:
  - name: kube-apiserver
contexts:
- context:
    cluster: authn
    user: kube-apiserver
  name: authn
current-context: authn
EOF
```

## Criar ficheiro de configuração do kubeadm

```bash
cat <<EOF >kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
apiServer:
  extraVolumes:
    - name: authentication-token-webhook-config-file
      mountPath: /etc/webhook-config.yaml
      hostPath: /root/webhook-config.yaml   
  extraArgs:
    authentication-token-webhook-config-file: /etc/webhook-config.yaml
  certSANs:
    - X.X.X.X       #IP address Kubernetes API server listens
networking:
  podSubnet: "10.244.0.0/16" #se usar o Flannel
EOF

```


## Converter ficheiro para uma versão mais recente e iniciar o Kubernetes com este mesmo ficheiro
```bash
kubeadm config migrate --old-config kubeadm-config.yaml --new-config kubeadm-config-new.yaml
kubeadm init --config kubeadm-config-new.yaml 
```


```bash
#Instalar um CNI plugin. 
#Exemplo o Flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```bash
#Criar as ClusterRole para os utilizadores ou grupos
```


# Client 

```bash
kubectl config set-credentials testuser \
 --token user:userpassword

kubectl config set-context user-context \
--cluster=kubernetes --user=user

kubectl config use-context  user-context

kubectl config set-cluster kubernetes  \
  --insecure-skip-tls-verify=true  \
  --server https://X.X.X.X:6443 

```

