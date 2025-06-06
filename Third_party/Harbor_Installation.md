### 1、安裝Harbor
> DNS需要正反解
```
root@harbor:~# curl -fsSL https://get.docker.com | sh
# Executing docker install script, commit: 53a22f61c0628e58e1d6680b49e82993d304b449
+ sh -c apt-get -qq update >/dev/null
.......略........

root@harbor:~# systemctl start docker && systemctl enable docker
Synchronizing state of docker.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable docker

root@harbor:~# apt install docker-compose -y
Reading package lists... Done
Building dependency tree... Done
.......略........

root@harbor:~# wget https://github.com/goharbor/harbor/releases/download/v2.12.3/harbor-offline-installer-v2.12.3.tgz
--2025-05-11 12:00:34--  https://github.com/goharbor/harbor/releases/download/v2.12.3/harbor-offline-installer-v2.12.3.tgz
Resolving github.com (github.com)... 20.27.177.113
Connecting to github.com (github.com)|20.27.177.113|:443... connected.
.......略........

root@harbor:~# tar xf harbor-offline-installer-v2.12.3.tgz -C /usr/local/src/

root@harbor:~# mkdir /usr/local/src/harbor/certs

root@harbor:~# cd /usr/local/src/harbor/certs

root@harbor:/usr/local/src/harbor/certs# openssl genrsa -out ca.key 4096

root@harbor:/usr/local/src/harbor/certs# openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=TP/L=TP/O=SmartX/OU=Lab/CN=harbor.andy.com" -key ca.key -out ca.crt

root@harbor:/usr/local/src/harbor/certs# openssl genrsa -out harbor.andy.com.key 4096

root@harbor:/usr/local/src/harbor/certs# openssl req -sha512 -new -subj "/C=CN/ST=TP/L=TP/O=SmartX/OU=Lab/CN=harbor.andy.com" -key harbor.andy.com.key -out harbor.andy.com.csr

root@harbor:/usr/local/src/harbor/certs#
cat > /usr/local/src/harbor/certs/v3.ext << EOF 
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1=harbor.andy.com
DNS.2=andy.com
DNS.3=harbor
EOF

root@harbor:/usr/local/src/harbor/certs# openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in harbor.andy.com.csr -out harbor.andy.com.crt
Certificate request self-signature ok
subject=C = CN, ST = TP, L = TP, O = SmartX, OU = Lab, CN = harbor.andy.com

root@harbor:/usr/local/src/harbor/certs# openssl x509 -inform PEM -in harbor.andy.com.crt -out harbor.andy.com.cert

root@harbor:/usr/local/src/harbor/certs# cd ..

root@harbor:/usr/local/src/harbor# cp harbor.yml.tmpl harbor.yml

root@harbor:/usr/local/src/harbor# vi harbor.yml
hostname: harbor.andy.com
http:
  port: 80
https:
  port: 443
  certificate: /usr/local/src/harbor/certs/harbor.andy.com.cert
  private_key: /usr/local/src/harbor/certs/harbor.andy.com.key

root@harbor:/usr/local/src/harbor# ./install.sh
[Step 5]: starting Harbor ...
[+] Running 10/10
 ✔ Network harbor_harbor        Created                                                                                         0.1s
 ✔ Container harbor-log         Started                                                                                         0.8s
 ✔ Container harbor-db          Started                                                                                         1.1s
 ✔ Container harbor-portal      Started                                                                                         1.1s
 ✔ Container registry           Started                                                                                         1.1s
 ✔ Container registryctl        Started                                                                                         1.1s
 ✔ Container redis              Started                                                                                         1.0s
 ✔ Container harbor-core        Started                                                                                         1.2s
 ✔ Container harbor-jobservice  Started                                                                                         1.5s
 ✔ Container nginx              Started                                                                                         1.5s
✔ ----Harbor has been installed and started successfully.----

# Web https://172.12.25.50/  ID:admin PW:Harbor12345 (預設)
```

### 2、Client安裝
```
[root@bastion ~]# scp root@172.12.25.50:/usr/local/src/harbor/certs/harbor.andy.com.cert ca.crt
The authenticity of host '172.12.25.50 (172.12.25.50)' can't be established.
.......略........

[root@bastion ~]# mkdir -p /etc/docker/certs.d/harbor.andy.com/

[root@bastion ~]# cp ca.crt /etc/docker/certs.d/harbor.andy.com/

[root@bastion ~]# yum install docker

[root@bastion ~]# yum install docker-compose-plugin.x86_64

[root@bastion ~]#
cat > /etc/docker/daemon.json << EOF
{
"insecure-registries" : ["harbor.andy.com:443", "0.0.0.0/0"]
}
EOF

[root@bastion ~]# systemctl daemon-reload

[root@bastion ~]# docker login harbor.andy.com -u admin -p Harbor12345
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Login Succeeded!
```
