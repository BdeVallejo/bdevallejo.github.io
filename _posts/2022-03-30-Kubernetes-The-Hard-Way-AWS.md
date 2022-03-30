---
layout: post
title: Kubernetes-The-Hard-Way con AWS
description: Ejercicio de configuración kubernetes de manera manual con AWS
tags: [ AWS , Kubernetes ]
---

# Indice

1. [Introducción](#1)

2. [Prerequisitos](#2)
    1. [Instalar AWS CLI](#2_1)

3. [Instalar las herramientas necesarias](#3) 
    1. [Instalar cfssl y cfssljson](#3_1)  
        1. [OS X](#3_1_1) 
        2. [Linux](#3_1_2) 
        3. [Verificación](#3_1_3)
    2. [Instalar kubectl](#3_2)  
        1. [OS X](#3_2_1) 
        2. [Linux](#3_2_2) 
        3. [Verificación](#3_2_3) 

4. [Configurar los recursos de AWS](#4) 
    1. [VPC](#4_1)
    2. [Subnet](#4_2) 
    3. [Internet Gateway](#4_3)
    4. [Tablas de enrutamiento](#4_4) 
    5. [Grupos de Seguridad](#4_5)
    6. [Balanceador de carga](#4_6)  

5. [Creación de las instancias](#5) 
    1. [Definición de la instancia](#5_1) 
    2. [Llaves ssh](#5_2)
    3. [Controller Nodes](#5_3) 
    4. [Woker Nodes](#5_4)

6. [Certificate Authority (CA)](#6)
    1. [Certificado de usuario admin](#6_2)
    2. [Kubelet Client Certificates](#6_2)
    3. [Certificado para el Controller Manager](#6_3)
    4. [Certificado para el kube-proxy](#6_4)
    5. [Certificado para el kube-scheduler](#6_5)
    6. [Certificado para el API Server](#6_6)
    7. [Service Account](#6_7)
    8. [Subiendo los certificados a los worker-nodes](#6_8)
    9. [Subiendo los certificados a los worker-nodes](#6_9)

7. [Generar Kubernetes Configuration Files para autenticarnos](#7)
    1. [Configurar la dirección del cluster](#7_1)
    2. [kubeconfigs para los kubelets](#7_2)
    3. [kubeconfigs para el kube-proxy](#7_3)
    4. [kubeconfig para el kube-controller-manager](#7_4)
    5. [kubeconfig para el kube-scheduler](#7_5)
    6. [kubeconfig para el usuario admin](#7_6)
    7. [Distribución de los kubeconfig a los worker-nodes y a los controller-nodes](#7_7)

8. [Encriptación de los datos en los Controller-Nodes](#8)
    1. [Generar una llave de encriptado](#8_1)
    2. [Archivo de configuración del encriptado](#8_2)

9. [Configuración del etcd](#9)
    1. [Bajar e instalar los binarios de etcd](#9_1)
    2. [Configuración del etcd server](#9_2)
    3. [Verificación de los etcd](#9_3)

10. [Configurando los Kubernetes Control Plane](#10)
    1. [Creando la carpeta config](#10_1)
    2. [Bajar e instalar los binarios del Kubernetes Controller](#10_2)
    3. [Configuración del Kubernetes API Server](#10_3)
    4. [Configuración del Kubernetes Controller Manager](#10_4)
    5. [Configuración del Kubernetes Scheduler](#10_5)
    6. [Poner en marcha el API Server, kube-controller-manager y kube-scheduler](#10_6)
    7. [Verificación](#10_7)
    8. [Añadir los hostnames de los worker-nodes](#10_8)

11. [Role-Base Access Control (RBAC)](#11)
    1. [Verificación del endpoint del cluster](#11_1)

12. [Configuración de los worker nodes](#12)
    1. [Instalar las dependencias](#12_1)
    2. [Deshabilitar Swap](#12_2)
    3. [Bajar e instalar los binarios](#12_3)
    4. [Configurar el CNI](#12_4)
    5. [Configurar containerd](#12_5)

13. [Configuración del kubelet](#13)
    1. [Configuración del proxy de Kubernetes](#13_1)
    2. [Iniciamos los servicios](#13_2)
    3. [Verificación](#13_3)

14. [Configuramos kubectl para poder acceder de manera remota](#14)
    1. [Configurando permisos para el usuario admin](#14_1)
    2. [Verificación](#14_2)

15. [Configurando la red de los Pods](#15)
    1. [Creando la tabla de enrutamiento](#15_1)
    2. [Validando las rutas](#15_2)

16. [Configurando el DNS add-on](#16)
    1. [Creando el DNS add-on](#16_1)
    2. [Verificación](#16_2)

17. [Probando nuestro clúster](#17)
    1. [Encriptación de los datos](#17_1)
    2. [Creando deployments](#17_2)
    3. [Redireccionamiento de puertos](#17_3)
    4. [Logs](#17_4)
    5. [Exec](#17_5)
    6. [Services](#17_6)

18. [Limpieza](#18)
    1. [Eliminar instancias](#18_1)
    2. [Eliminar el balanceador de carga, el gateway y los demás recursos de la red](#18_2)

# Introducción <a id="1"></a>

Estoy preparando la certificación [CKA ( Certified Kubernetes Administrator )](https://www.cncf.io/certification/cka/) y para ello hoy voy a  configurar un clúster de Kubernetes de manera manual desde cero para entender cómo funciona. Para ello voy a seguir la guía [Kubernetes the Hard Way AWS de Prabhatsharma](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws) que a su vez está basada en la [Kubernetes the Hard Way de Kelseyhightower.](https://github.com/kelseyhightower/kubernetes-the-hard-way)

Todos los comandos están copiados de estas guías, lo único que he hecho en este artículo es desgranarlos para entender qué voy haciendo con cada uno de ellos y traducir algunas cosas de la guía original. Así que recomiendo tener como referencia la guía de  Prabhatsharma o bien la de Kelseyhightower, ya que son las más actualizada y originales. Recomiendo, eso sí, leer esta artículo si te pierdes por el camino o bien no entiendes alguno de los pasos.

# Prerequisitos <a id="2"></a>

La parte de prerequisitos y la siguiente están explicadas en el artículo [Trabajando con AWS mediante CLI](https://bdevallejo.github.io/2022/03/11/Lanzar-instancias-AWS-mediante-CLI/). Así que pasaré por esta parte un poco de puntillas. Pero cuidado, los comandos son ligeramente diferentes:

## Instalar AWS CLI <a id="2_1"></a>

Aquí puedes ver la  [documentación](https://aws.amazon.com/cli/). Una vez instalada puedes verificarlo con: 

```aws --version```.

Configuramos una región por defecto (eu-central-1 en mi caso):

```
AWS_REGION=eu-central-1
aws configure set default.region $AWS_REGION
```

La guía nos anima a instalar [tmux](https://github.com/tmux/tmux/wiki) para facilitarnos algunos pasos. Pero no es necesario.

# Instalar las herramientas necesarias <a id="3"></a>

## cfssl y cfssljson <a id="3_1"></a>

[cfssl y cfssljson](https://github.com/cloudflare/cfssl) son dos herramientas que nos ayudan a generar una PKI (Public Key Infrastructure), es decir, a generar certificados digitales así como un conjunto de reglas que luego utilizaremos para dar seguridad a los diferentes componentes de nuestro cluster. Dicho así suena muy teórico pero en seguida cobra sentido.

### OS X <a id="3_1_1"></a>

```
curl -o cfssl https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/darwin/cfssl
curl -o cfssljson https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/darwin/cfssljson
```
```
chmod +x cfssl cfssljson
```


```
sudo mv cfssl cfssljson /usr/local/bin/
```
Si no funciona lo anterior, usa Homebrew
```
brew install cfssl
```

### Linux <a id="3_1_2"></a>

```
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
```


```
chmod +x cfssl cfssljson
```


```
sudo mv cfssl cfssljson /usr/local/bin/
```

### Verificación <a id="3_1_3"></a>

Es importante que las versiones de cfssl y cfssljson sean 1.4.1 o superiores:
```
cfssl version
```

```
cfssljson --version
```

## Instalar kubeclt <a id="3_2"></a>

Kubectl es el CLI que se utiliza para comunicarnos con el API Server de Kubernetes. Si has usado Kubernetes alguna vez, sobran las explicaciones. Es la herramienta que utilizamos para casi todo. 

### OS X <a id="3_2_1"></a>

```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Linux <a id="3_2_2"></a>

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
```


```
chmod +x kubectl
```


```
sudo mv kubectl /usr/local/bin/
```

### Verificación <a id="3_2_3"></a>

Es importante contar con la versión 1.21.0 o superior:
```
kubectl version --client
```

# Configurar los recursos de AWS <a id="4"></a>

Esta parte está también explicada en su práctica totalidad en el artículo trabajando con AWS mediante CLI. Por tanto sólo explico lo demás.

## VPC <a id="4_1"></a>

Configuramos la VPC (Virtual Private Cloud): 

```
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output text --query 'Vpc.VpcId')
aws ec2 create-tags --resources ${VPC_ID} --tags Key=Name,Value=kubernetes-the-hard-way
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support '{"Value": true}'
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames '{"Value": true}'
```

## Subnet <a id="4_2"></a>

```
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.0.1.0/24 \
  --output text --query 'Subnet.SubnetId')
aws ec2 create-tags --resources ${SUBNET_ID} --tags Key=Name,Value=kubernetes
```

## Internet Gateway <a id="4_3"></a>

```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags --resources ${INTERNET_GATEWAY_ID} --tags Key=Name,Value=kubernetes
aws ec2 attach-internet-gateway --internet-gateway-id ${INTERNET_GATEWAY_ID} --vpc-id ${VPC_ID}
```

## Tablas de enrutamiento <a id="4_4"></a>


```
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags --resources ${ROUTE_TABLE_ID} --tags Key=Name,Value=kubernetes
aws ec2 associate-route-table --route-table-id ${ROUTE_TABLE_ID} --subnet-id ${SUBNET_ID}
aws ec2 create-route --route-table-id ${ROUTE_TABLE_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${INTERNET_GATEWAY_ID}
```

## Grupos de Seguridad <a id="4_5"></a>

``` 
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name kubernetes \
  --description "Kubernetes security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')
aws ec2 create-tags --resources ${SECURITY_GROUP_ID} --tags Key=Name,Value=kubernetes
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.200.0.0/16
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 6443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol icmp --port -1 --cidr 0.0.0.0/0
```

## Balanceador de carga <a id="4_6"></a>

Creamos un balanceador que decidirá a dónde dirigirnos según el tráfico y la carga de las diferentes instancias. Asimismo definimos un target group, que básicamente le dice al balanceador de carga a dónde tiene que dirigir el tráfico (en este caso le damos tres IPs que corresponden a los controller-nodes). 

```
 LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
    --name kubernetes \
    --subnets ${SUBNET_ID} \
    --scheme internet-facing \
    --type network \
    --output text --query 'LoadBalancers[].LoadBalancerArn')
  TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
    --name kubernetes \
    --protocol TCP \
    --port 6443 \
    --vpc-id ${VPC_ID} \
    --target-type ip \
    --output text --query 'TargetGroups[].TargetGroupArn')
  aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets Id=10.0.1.1{0,1,2}
  aws elbv2 create-listener \
    --load-balancer-arn ${LOAD_BALANCER_ARN} \
    --protocol TCP \
    --port 443 \
    --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
    --output text --query 'Listeners[].ListenerArn'
```


```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
```

# Creación de las instancias <a id="5"></a>

## Definición de la instancia <a id="5_1"></a>

Definimos el tipo de instancia que queremos (Ubuntu en este caso)

```
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --output json \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
```


## Llaves ssh <a id="5_2"></a>

Creamos un key pair y lo llamamos “kubernetes”. Lo utilizaremos para crear los diferentes nodos, tanto los controllers como los worker-nodes, y después poder conectarnos a ellos.  También cambiamos los permisos para poder ejecutarlo:

```
aws ec2 create-key-pair --key-name kubernetes --output text --query 'KeyMaterial' > kubernetes.id_rsa
chmod 600 kubernetes.id_rsa
```

## Controller Nodes <a id="5_3"></a>

NOTA: Me refiero en toda la documentación a estos nodos como controller nodes. No he encontrado una traducción al castellano mejor.

Primeramente creamos 3 nodos que harán las veces de controllers con la imagen que hemos definido antes. Se llamarán controller-0, controller-1 y controller-2. Y sus IPs privadas serán 10.10.1.10, 10.10.1.11 y 10.10.1.12 respectivamente.

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t3.micro \
    --private-ip-address 10.0.1.1${i} \
    --user-data "name=controller-${i}" \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=controller-${i}"
  echo "controller-${i} created "
done
```

## Worker Nodes <a id="5_4"></a>

NOTA: Me refiero a estos nodos como worker nodes en toda la documentación. De la misma manera que uso el término controller nodes, no he encontrado una mejor traducción al castellano.

Y ahora creamos 3 worker nodes. Se llamarán worker-0, worker-1 y worker-2. Y sus IPs privadas serán 10.10.1.20, 10.10.1.21 y 10.10.1.22 respectivamente.

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t3.micro \
    --private-ip-address 10.0.1.2${i} \
    --user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=worker-${i}"
  echo "worker-${i} created"
done
```

# Certificate Authority (CA) <a id="6"></a>

Como he dicho anteriormente, debemos generar una PKI (Public Key Infrastructure), es decir, generar certificados digitales TLS para todos los diferentes componentes de nuestro cluster (etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, y kube-proxy). 

Todos ellos deben cumplir con unas reglas de seguridad para poder comunicarse entre ellos, y esto pasa por crear y almacenar certificados válidos.

Para generar los certificados utilizamos cfssl, que es la herramienta de [CloudFlare](https://blog.cloudflare.com/introducing-cfssl/), que nos facilita un poco el proceso.

Este paso es algo repetitivo, ya que básicamente definimos los certificados en un formato json. [Este enlace puede servir para entender mejor el formato y sus parámetros](https://chowdera.com/2020/11/202011251127429414.html). Después generaremos un certificado y una llave privada con cfssl y cfssljson. 

Lo primero es generar un Certificate Authority (CA) que nos sirve para firmar todos los certificados TLS. 

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "L": "Madrid",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Spain"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Así obtenemos dos archivos: `ca-key.pem` y `ca.pem`. Con ellos podemos firmar los certificados siguientes:

## Certificado de usuario admin <a id="6_1"></a>

Ahora generamos un certificado y una llave privada para el administrador (usuario admin). Es necesario especificar que el usuario admin está dentro del grupo “system:masters” (ver el parámetro “O”  dentro de “names): 

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "L": "Madrid",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "OU": "CA",
      "ST": "Spain"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Así obtenemos dos archivos: `admin-key.pem` y  `admin.pem`.

##  Kubelet Client Certificates<a id="6_2"></a>

Los kubelets son los responsables de los nodos, sus pods y containers. Para que puedan comunicarse con el API Server se utiliza un tipo de autorización llamada [Node Authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/). Estos certificados han de tener un identificador que los defina como miembros del grupo system:nodes y cuyo nombre de usuario tiene el formato “system:node:nombredelnodo”  (mira el CN o Common Name y el parámetro O dentro de names).

```
for i in 0 1 2; do
  instance="worker-${i}"
  instance_hostname="ip-10-0-1-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
       "C": "ES",
      "L": "Madrid",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Spain"
    }
  ]
}
EOF

  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  internal_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PrivateIpAddress')

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance_hostname},${external_ip},${internal_ip} \
    -profile=kubernetes \
    worker-${i}-csr.json | cfssljson -bare worker-${i}
done
```

Como hay tres worker-nodes habremos generado 3 certificados y 3 keys para cada uno de los kubelets correspondientes a cada nodo.

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

## Certificado para el Controller Manager<a id="6_3"></a>

Seguimos creando el certificado y la key para el kube-controller-manager. 

```
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "L": "Madrid",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Spain"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

## Certificado para kube-proxy<a id="6_4"></a>

Generamos el certificado y la key para el kube:proxy.

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "L": "Madrid",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Spain"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

## Certificado del kube-scheduler<a id="6_5"></a>

Seguimos creando el certificado y la key para el kube-scheduler-manager. 

```
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "L": "Madrid",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Spain"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```


## Certificado del API Server<a id="6_6"></a>

Creamos ahora el certificado y la key para el API Server.

El API Server de Kubernetes usa certificados para encriptar el tráfico entrante y saliente así como para comprobar la procedencia de las conexiones que le llegan. Por otra parte, para conectarnos al API Server normalmente utilizaremos una herramienta como kubectl y nos dirigiremos a él mediante una IP o bien un hostname. Por tanto, hemos de asegurarnos que creamos una lista con todos los hostnames e IPs con las que identificamos al API Server. Esta lista se llama Subject Alternative Names (SANs). Si nos intentamos conectar al API Server con un nombre diferente, se nos aparecerá un error diciendo que el certificado que tenemos no es válido.

Para ello primero creamos una lista para asignarle diferentes nombres: 

```
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local
```

Y ahora creamos el certificado:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "L": "Madrid",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Spain"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.0.1.10,10.0.1.11,10.0.1.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

La dirección 10.32.0.1 que hemos definido como hostname es la primera dirección del bloque de direcciones IP 10.32.0.0/24 que vamos a reservar para los service-cluster durante la configuración del [API Server](#10_3), en la sección de configuración del Control Plane. 


## Service Account<a id="6_7"></a>

Pongamos por ejemplo que tenemos una web en el cluster de Kubernetes que tiene que hacer alguna tarea mediante la API (por ejemplo, trabajar con otros pods o cualquier otra cosa). Para identificarse necesitará certificados y permisos. Para ello hay que crear un Service Account. Como pone en la documentación oficial, "las User Accounts son para humanos y las Service Accounts son para procesos que se ejecutan en los pods". 


Crearemos una key que a su vez podrá generar token para las diferentes Service Accounts.

```
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "L": "Madrid",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Spain"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

## Subir los Certificados a los Worker-Nodes<a id="6_8"></a>

Mediante scp (Secure Copy), copiamos los Certificate Authority y los certificados del servidor (los que hemos generados en la sección [Kubelet Client Certificates](#6_2)) a los worker-nodes:

```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/
done
```

## Subir los certificados a los Controller-Nodes<a id="6_8"></a>

Subimos el Certificate Authority, los certificados del API Server y los Service Account a los Controller-Nodes. El resto de certificados los iremos distribuyendo más adelante, ya que los vamos a necesitar para generar archivos de configuración. 

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ubuntu@${external_ip}:~/
done
```

# Generar Kubernetes Configuration Files para autenticarnos<a id="7"></a>

Como hemos visto más arriba, para comunicarnos con un cluster lo haremos mediante el API Server. Y para ello tenemos que generar un archivo que contenga la información acerca de los diferentes clusters, usuarios, namespaces y mecanismos de autentificación. Esos archivos se llaman `kubeconfig` y suelen estar localizados en el directorio `$HOME/.kube’. 

Importante: En esta parte vamos a utilizar los certificados que hemos generado anteriormente. Por tanto, hemos de trabajar en el mismo directorio en el que estábamos.

## Configurar la dirección del cluster <a id="7_1"></a>

Primeramente vamos a configurar la dirección IP que utilizaremos para conectarnos en la API Server y que incluiremos en nuestros archivos kubeconfig. Esta dirección es la del Balanceador de Carga o Load Balancer que hemos configurado en AWS anteriormente.

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[0].DNSName')
```

## kubeconfigs para los kubelets<a id="7_2"></a>

Como hemos visto en la sección de [Kubelet Client Certificates](#6_2), para que los kubelets puedan ser reconocidos y autorizados para comunicarse con el API Server, se utiliza un tipo de autorización especial llamada Node Authorizer. Para ello, había que darles un nombre con el formato "system:node:nombredelnodo". Esta información ha de ser incluída en los archivos kubeconfig de los diferentes kubelets para que puedan identificarse, así como los certificados apropiados.

```
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

Se habrán generado tres archivos, uno para cada kubelet (es decir, para cada worker-node):

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```


## kubeconfig para el kube-proxy <a id="7_3"></a>

Generaremos ahora el archivo kubeconfig para el kube-proxy:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```


## kubeconfig para el kube-controller-manager<a id="7_4"></a>

Lo siguiente es generar ahora el kubeconfig para el kube-controller-manager:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```


## kubeconfig para el kube-scheduler<a id="7_5"></a>

Haremos lo mismo para el kube-scheduler:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

## kubeconfig para el usuario admin<a id="7_6"></a>

Por último generamos un kubeconfig para el usuario admin:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

## Distribución de los kubeconfig a los worker-nodes y a los controller-nodes<a id="7_7"></a>

Primeramente vamos a copiar los kubeconfig de los kubelet y del kube-proxy a los diferentes worker-nodes mediante scp (Secure Copy): 

```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    ${instance}.kubeconfig kube-proxy.kubeconfig ubuntu@${external_ip}:~/
done
```

Después haremos lo mismo con los kubeconfig del kube-controller-manager y del kube-scheduler, además del kubeconfig del usuario administrador (o admin) pero estos serán copiados a los controller-nodes :

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  
  scp -i kubernetes.id_rsa \
    admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@${external_ip}:~/
done
```

# Encriptación de los datos en los Controller-Nodes<a id="8"></a>

Por seguridad vamos a encriptar los datos y secretos almacenados en nuestro cluster. Para ello vamos a utilizar una llave de encriptado y a definir un archivo de configuración que luego subiremos a los Controller-Nodes.

## Generar una llave de encriptado<a id="8_1"></a>

El primer paso es simplemente generar una llave de encriptado con [/dev/urandom](https://linux.die.net/man/4/urandom) para luego codificarla en base64.

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## Archivo de configuración del encriptado<a id="8_2"></a>

Ahora crearemos un archivo de configuración. Para ello utilizamos el algoritmo [AESCBC (Advanced Encryption Standard Cipher-Block Chaining)](https://www.educative.io/edpresso/what-is-cbc) que a su vez tomará como parámetro la llave que acabamos de generar.  

Para más información sobre este paso o para utilizar otro algoritmo ver la página oficial de kubernetes. Por ejemplo, si vemos la [parte de algoritmos](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/), nos dice que aescbc no se recomienda portque CBC tiene una vulnerabilidad y se recomienda utilizar KMS en su lugar. Sin embargo, dado que es un ejercicio para aprender, voy a seguir los pasos de la guía tal y como están.

Esta llave `key1`la utilizaremos más adelante durante la fase de pruebas para verificar la encriptación. 

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Ahora copiaremos este archivo en cada uno de los controllers

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  
  scp -i kubernetes.id_rsa encryption-config.yaml ubuntu@${external_ip}:~/
done
```

# Configuración del etcd<a id="9"></a>

El etcd es básicamente un almacén de datos. Para ejecutarlo sólo hay que bajar el binario e instalarlo. Por defecto funciona mediante el puerto 2379 y viene con un controlador, que podemos usar con el comando `etcdctl`.

En un entorno de alta disponibilidad tendremos varios etcd corriendo en el puerto 2379. Por ello hay que hacer que se conozcan entre ellos y que decidan quién de ellos lleva la copia maestra de los datos. Por así decirlo, eligen un etcd líder.  

En este caso, vamos a crear tres etcds, uno por cada uno de los controller-nodes. Para ello, el paso que nos recomienda realizar la guía primero es imprimir en consola el comando para conectarnos mediante ssh a cada uno de los controllers. Empezaremos configurando el etcd en el controller-0, luego nos conectaremos al controller-1 y luego al controller-2. La guía recomienda trabajar en este proceso con `tmux`, ya que nos facilita el realizar varias tareas a la vez. Pero no es obligatorio.

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  echo ssh -i kubernetes.id_rsa ubuntu@$external_ip
done
```

## Bajar e instalar los binarios de etcd<a id="9_1"></a>

Una vez hemos hecho ssh a uno de los controllers, bajaremos el binario:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```

Lo extraemos y lo instalamos:

```
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
```

## Configuración del etcd server<a id="9_2"></a>

Ahora creamos dos directorios. En uno de ellos copiaremos el Certificate Autority (CA), y  certificado y llave del API Server para que el etcd pueda encontrarlos y etcdctl pueda trabajar con el API Server. 

```
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

Ahora crearemos una variable llamada INTERNAL_IP que servirá para comunicarnos con los diferentes etcd ( también conocidos como etcd cluster peers ). Para ello utilizaremos la “IP Mágica” de AWS (http://169.254.169.254) que sirve para obtener datos del usuario así como metadatos de una instancia. Obviamente esta IP sólo funciona desde dentro del host, ya que de lo contrario estaríamos bastante expuestos.

```
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

Y mediante la misma dirección pero esta vez apuntando al directorio `/user-data` obtenemos un identificador único para cada uno de los etcd. 

```
ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${ETCD_NAME}"
```

Con todo esto y con los certificados en el directorio `/etc/etcd/ ` ya podemos crear el archivo de configuración del etcd.

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.0.1.10:2380,controller-1=https://10.0.1.11:2380,controller-2=https://10.0.1.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Ahora arrancamos el etcd:

```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

Y tras hacer todo esto tenemos que loguearnos del controller-0 y hacer ssh al controller-1 y luego al controller-2. Cuando hayamos terminado de configurar los tres podemos seguir con el siguiente paso.

## Verificación de los etcd<a id="9_3"></a>

Verificamos que los etcd se hayan configurado correctamente y estén funcionando.

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

Deberíamos obtener en consola algo como esto:

```
bbeedf10f5bbaa0c, started, controller-2, https://10.0.1.12:2380, https://10.0.1.12:2379, false
f9b0e395cb8278dc, started, controller-0, https://10.0.1.10:2380, https://10.0.1.10:2379, false
eecdfcb7e79fc5dd, started, controller-1, https://10.0.1.11:2380, https://10.0.1.11:2379, false
```

# Configurando los Kubernetes Control Plane<a id="10"></a>

En esta sección vamos a configurar los Control Plane en los nodos que hemos venido llamando controller-0, controller-1 y controller-2.

Primeramente vamos a ejecutar un comando similar al que hemos ejecutado en la sección anterior simplemente para imprimir en consola cómo nos debemos conectar con ssh a cada uno de los controller-nodes. Si ya estás conectado a los controllers con tmux obviamente puedes saltar este paso. 

´´´
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  echo ssh -i kubernetes.id_rsa ubuntu@$external_ip
done
´´´

Empezaremos configurando el controller-0, luego nos conectaremos al controller-1 y luego al controller-2. La guía recomienda trabajar en este proceso con `tmux` al igual que en la sección anterior para agilizarlo, pero no es obligatorio. 

## Creando la carpeta config<a id="10_1"></a>

Vamos a crear primeramente un directorio donde podremos copiar los archivos de configuración del kube-scheduler. 
sudo mkdir -p /etc/kubernetes/config

## Bajar e instalar los binarios del Kubernetes Controller<a id="10_2"></a>

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubect"

```

Una vez bajados, instalamos los binarios:

```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

## Configuración del Kubernetes API Server<a id="10_3"></a>

Creamos una carpeta donde moveremos los certificados apropiados (el Certificate Authority y el certificado y llave del API Server, del Service Account y el archivo de configuración para la encriptación de los datos). 

```
sudo mkdir -p /var/lib/kubernetes/
```

```
sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

Tal y como hemos hecho con los etcd, crearemos una variable llamada INTERNAL_IP exponer la API Server a los diferentes componentes del cluster. Para ello utilizaremos de nuevo la “IP Mágica” de AWS (http://169.254.169.254) 

```
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

Con esto ya podemos crear el archivo de configuración de tipo service unit llamado kube-apiserver.service. Para ver más info sobre las diferentes unidades del sistema sytemd y los archivos que vamos a crear a continuación podemos ver este [link](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files).

Para más información sobre los diferentes parámetros que utilizamos en el comando siguiente, ver este otro [link](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)


```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.0.1.10:2379,https://10.0.1.11:2379,https://10.0.1.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Tal y como vimos cuando creamos el [certificado del API Server](#6_6), la dirección 10.32.0.1 que definimos como uno de los hostnames es la primera dirección del bloque de direcciones IP 10.32.0.0/24 que hemos asignado a los service-cluster (ver la flag --service-cluster-ip-range). 

## Configuración del Kubernetes Controller Manager<a id="10_4"></a>

Primeramente moveremos el kubeconfig que hemos creado anteriormente a la carpeta correspondiente (`/var/lib/kubernetes `):

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Y ahora creamos otro archivo del tipo service unit:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## Configuración del Kubernetes Scheduler<a id="10_5"></a>

Haremos lo mismo con el kube-scheduler. Primero movemos el kubeconfig a su sitio:

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

En un entorno de alta disponibilidad como este tenemos varios schedulers, pero es importante recordar que sólo un scheduler puede actuar al tiempo. 

Por tanto, al igual que hemos hecho con los etcd, hay que elegir un líder scheduler. Así que primero creamos un yaml donde especificamos quién es el scheduler líder, con el parámetro –leader-elect=true. 

```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Y ahora crearemos el archivo de tipo systemd service unit:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## Poner en marcha el API Server, kube-controller-manager y kube-scheduler<a id="10_6"></a>

Una vez hecho esto podemos poner en marcha los servicios corrrespondientes:

```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

Dejaremos unos 10 segundos para que se inicien los procesos correspondientes.

## Verificación<a id="10_7"></a>

Podemos verificar que todo funciona con:

```
kubectl cluster-info --kubeconfig admin.kubeconfig
```

Lo cual debería devolver algo como:

```
Kubernetes control plane is running at https://127.0.0.1:6443
```

Que certifica que podemos conectarnos mediante el “puerto seguro”(https://kubernetes.io/es/docs/concepts/security/controlling-access/#puertos-e-ips-del-api-server) 6443

## Añadir los hostnames de los worker-nodes<a id="10_8"></a>

Para poder ejecutar comandos con kubectl, tenemos que indicar a los controller-nodes cuáles son los hostnames de los worker-nodes. Así que simplemente añadiremos tres líneas a /etc/hosts con:

```
cat <<EOF | sudo tee -a /etc/hosts
10.0.1.20 ip-10-0-1-20
10.0.1.21 ip-10-0-1-21
10.0.1.22 ip-10-0-1-22
EOF
```

De lo contrario en la sección DNS Cluster obtendremos un error como este: `Error from server: error dialing backend: dial tcp: lookup ip-10-0-1-22 on 127.0.0.53:53: server misbehaving`.

Repetiremos todo este proceso para cada uno de los controllers (controller-0, controller-1 y controller-2).

# Role-Base Access Control (RBAC) <a id="11"></a>

Más adelante, durante la configuración de los kubelets, configuraremos el modo de acceso (authorization-mode) al tipo Webhook. Webhook comprueba el acceso de cualquier usuario que quiera interactuar con él, y tal y como aparece en la documentación de kubernetes, este método [es útil para delegar la comprobación de permisos al API Server](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access).

Dicho de otra forma, tal y como vamos a configurar nuestros worker-nodes, para que el API Server pueda interactuar con los kubelets de los worker-nodes y así ejecutar comandos y tener acceso a logs, métricas, etcétera; tenemos que garantizar su acceso mediante un role-base access control.

En este caso basta con ejecutar los comandos desde cualquiera de los controllers. Vamos a imprimir en consola el comando para conectarnos con ssh al controller-0 pero si ya estamos en cualquiera de los otros funcionaría igual.

```
external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=controller-0" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip}
```

Ahora crearemos un ClusterRole llamado system:kube-apiserver-to-kubelet con suficientes permisos para acceder al kubelet y ejecutar diferentes tareas:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Ahora vamos a crear un ClusterRoleBinding que autoriza al API Server a conectarse al Kubelet. Es importante recordar que el API Server se va conectar como si fuera un usuario con el nombre kubernetes, tal y como hemos configurado en la sección “Configuración del Kubernetes API Server” mediante la flag --kubelet-client-certificate. Así que sólo queda unir el ClusterRole que acabamos de crear (system:kube-apiserver-to-kubelet) con el usuario kubernetes.
```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

```

## Verificación del endpoint del cluster<a id="11_1"></a>

NOTA: En la [guia oficial](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/03-compute-resources.md) hay una nota que nos avisa que tenemos que ejecutar los comandos siguientes desde la misma máquina con la que hemos creado las instancias de aws. Sin embargo, en mi caso seguía sin funcionar. Por tanto, tuve que añadir el parámetro [-k al comando](https://linuxpip.org/curl-failed-to-verify-the-legitimacy-of-the-server/).

Para verificar que todo funciona, debemos ejecutar estos comandos desde el mismo directorio donde hemos creado nuestro Certificate Authority. Primero vamos a obtener la dirección del balanceador de carga: 

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
```

Y luego haremos un HTTP request para ver la versión de Kubernetes (aquí es donde añado el parámetro -k que deshabilita la comprobación estricta del certificado:

```
curl -k --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}/version
```

Lo cual debería darnos un resultado similar a este:

```
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.0",
  "gitCommit": "cb303e613a121a29364f75cc67d3d580833a7479",
  "gitTreeState": "clean",
  "buildDate": "2021-04-08T16:25:06Z",
  "goVersion": "go1.16.1",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

# Configuración de los worker nodes<a id="12"></a>

Ahora vamos a pasar a configurar los worker nodes. Además instalaremos runc (una herramienta CLI para trabajar con contenedores), container networking plugins para configurar la conexión de los contenedores y además liberar recursos cuando un contenedor es eliminado, y containerd, que es el runtime de Linux. 

Una vez más, para tener el comando con el que nos conectaremos a cada uno de los worker-nodes utilizaremos este pequeño script:

```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  echo ssh -i kubernetes.id_rsa ubuntu@$external_ip
done
```

También habrá que conectarse a cada uno de los worker-nodes y ejecutar todos los pasos. Para ello, se recomienda usar `tmux`que agilizará el proceso al poder realizar tareas en paralelo. 

## Instalar las dependencias<a id="12_1"></a>

Primero instalaremos las dependencias necesarias en el sistema operativo
(socat es necesario para más adelante poder ejecutar el port-forwarding de kubectl).

```
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
```

## Deshabilitar Swap<a id="12_2"></a>

[Swap](https://help.ubuntu.com/community/SwapFaq) es una parte de la memoria virtual de una máquina que está reservada por si hay procesos que necesitan recursos adicionales para poder ejecutarse. Dicho de otra forma, es como una memoria de emergencia.

Si tenemos swap configurado en nuestro sistema operativo, el kubelet no podrá ejecutarse, por lo que primero vamos a verificar si está activado:

```
sudo swapon --show
```

Si en la consola no aparece nada significa que podemos continuar. De lo contrario, ejecutaremos el siguiente comando:

```
sudo swapoff -a
```

La guía recomienda comprobar la documentación de Linux para asegurarse de que swap permanecerá desactivado si hacemos un reboot de la máquina. Este [enlace puede ser de utilidad](https://www.geeksforgeeks.org/how-to-permanently-disable-swap-in-linux/)

## Bajar e instalar los binarios<a id="12_3"></a>

Empezaremos instalando y bajando los binarios necesarios:

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet
```

Antes de instalarlos debemos crear los directorios apropiados:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Y ya podemos instalarlos:

```
mkdir containerd
tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
sudo mv runc.amd64 runc
chmod +x crictl kubectl kube-proxy kubelet runc 
sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
sudo mv containerd/bin/* /bin/
```

## Configurar el CNI<a id="12_4"></a>

CNI (Container Network Interface), es un  conjunto de especificaciones y librerías para configurar interfaces de red en contenedores. Dicho de otra forma, es un estándar de ejecutar todas las tareas necesarias para crear una red. Como ya hemos instalado container networking plugins y creado una carpeta `/etc/cni/net.d` ahora simplemente vamos a configurar la red y guardar esta configuración en la carpeta.

Recomiendo la lectura de este [artículo](https://www.digitalocean.com/community/tutorials/kubernetes-networking-under-the-hood) que explica la arquitectura de red dentro de un cluster y cómo se comunican los Pods entre sí, ya que esta parte puede resultar algo confusa y compleja.

Primeramente tomamos el bloque de direcciones que hemos configurado en los primeros pasos, cuando creábamos las instancias de los worker-nodes. Ahí definimos el parámetro `--user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24"` donde reservamos un bloque de direcciones en cada uno de los nodos para los Pods. 

```
POD_CIDR=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^pod-cidr" | cut -d"=" -f2)
echo "${POD_CIDR}"
```

Una vez tenemos el bloque de direcciones IP podemos crear el configuration file para crear la `bridge network` dentro de cada nodo y que los pods puedan comunicarse entre sí.

```
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Ahora haremos lo mismo para crear la loopback configuration file:

```
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

# Configurando containerd<a id="12_5"></a>


ContainerD es el runtime de Linux y Windows, que se encarga de administrar el ciclo de vida del sistema.

Creamos el containerd configuration file, no sin antes haber creado el directorio apropiado:

```
sudo mkdir -p /etc/containerd/
```

```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

Y ahora creamos el unit file containerd.service:

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

# Configuración del kubelet<a id="13"></a>

Para configurar el kubelet primero tenemos que mover los certificados y las llaves que hemos copiado previamente a su carpeta correspondiente.

```
WORKER_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
| tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${WORKER_NAME}"
```

```
sudo mv ${WORKER_NAME}-key.pem ${WORKER_NAME}.pem /var/lib/kubelet/
sudo mv ${WORKER_NAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

Creamos el archivo de configuración kubelet-config.yaml:

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}-key.pem"
EOF
```

Cabe resaltar que el parámetro resolvConf se usa para evitar bucles cuando utilizamos CoreDNS para descubrir servicios en máquinas que utilizan systemd-resolved (systemd-resolved es un servicio que básicamente proporciona resolución DNS a aplicaciones locales)

Ahora crearemos el archivo kubelet.service:

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## Configuración del proxy de Kubernetes<a id="13_1"></a>

Ya sólo nos falta configurar el kube-proxy, que actúa como un proxy (valga la redundancia) dentro de cada nodo y se encarga de que los Pods se puedan comunicar entre sí.

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Creamos el kube-proxy-config.yaml:

```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

Y por último el kube-proxy.service systemd unit file:

```

cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## Iniciamos los servicios<a id="13_2"></a>

Iniciamos los servicios que acabamos de configurar:

```
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

Si todo está correcto, no debería aparecer ningún mensaje. Pero por si acaso vamos a verificarlo. 

## Verificación<a id="13_3"></a>

NOTA: En la [guia oficial](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/03-compute-resources.md) hay una nota que nos avisa que tenemos que ejecutar los comandos siguientes desde la misma máquina con la que hemos creado las instancias de aws.

Nos desconectamos de las instancias y listamos los nodos con:

```
external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=controller-0" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip} kubectl get nodes --kubeconfig admin.kubeconfig
```

Lo cual debería mostrar algo como esto:

```
NAME             STATUS   ROLES    AGE   VERSION
ip-10-0-1-20   Ready    <none>   51s   v1.21.0
ip-10-0-1-21   Ready    <none>   51s   v1.21.0
ip-10-0-1-22   Ready    <none>   51s   v1.21.0
```

# Configuramos kubectl para poder acceder de manera remota<a id="14"></a>

Vamos a proceder a configurar kubectl para poder acceder de manera remota.

## Configurando permisos para el usuario admin<a id="14_1"></a>

Vamos a utilizar la dirección IP del balanceador de carga como puerta de acceso a los API Server. Después con los certificados apropiados vamos a crear un contexto llamado kubernetes-the-hard-way y a permitir el acceso al usuario admin.

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
--load-balancer-arns ${LOAD_BALANCER_ARN} \
--output text --query 'LoadBalancers[].DNSName')

kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

kubectl config use-context kubernetes-the-hard-way
```

## Verificación<a id="14_2"></a>

Comprobamos primeramente la versión de kubectl:

```
kubectl version
```

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

Y ahora comprobamos si podemos ver los nodos:

```
kubectl get nodes
```

```
NAME           STATUS   ROLES    AGE     VERSION
ip-10-0-1-20   Ready    <none>   3m35s   v1.21.0
ip-10-0-1-21   Ready    <none>   3m35s   v1.21.0
ip-10-0-1-22   Ready    <none>   3m35s   v1.21.0
```

# Configurando la red de los Pods<a id="15"></a>

Tal y como hemos configurado antes, los diferentes pods van a recibir una IP del rango `10.200.0.0/24` para el worker-0, `10.200.1.0/24` para el worker-1 y `10.200.2.0/24` para el worker-2 (lo que hemos llamado Pod CIDR range). 

Ahora vamos a configurar los nodos de manera que los pods puedan comunicarse entre sí. Normalmente utilizaríamos herramientas como  flannel, calico o amazon-vpc-cni-k8s pero en este ejercicio se trata de hacer todo de manera manual para entenderlo mejor.

## Creando la tabla de enrutamiento<a id="15_1"></a>

Vamos a empezar creando la Routing Table. 

NOTA: En el siguiente comando hay un parámetro `ROUTE_TABLE_ID` que hemos asignado casi al principio del ejercicio. En mi caso tardé más de un día en hacer todo, ya que el escribir todas las notas y entender todo conlleva su tiempo. Si aparece un mensaje de error en el que te dice que RouteTableId es un parámetro necesario o algo parecido, tan sólo has de volver a guardar el parámetro con:

```
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables –filters “Name=tag:Name, Values=”kubernetes”” –output=text –query ‘RouteTables[].RouteTableId’)
```

Una vez aclarado, podemos continuar:

```
for instance in worker-0 worker-1 worker-2; do
  instance_id_ip="$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]')"
  instance_id="$(echo "${instance_id_ip}" | cut -f1)"
  instance_ip="$(echo "${instance_id_ip}" | cut -f2)"
  pod_cidr="$(aws ec2 describe-instance-attribute \
    --instance-id "${instance_id}" \
    --attribute userData \
    --output text --query 'UserData.Value' \
    | base64 --decode | tr "|" "\n" | grep "^pod-cidr" | cut -d'=' -f2)"
  echo "${instance_ip} ${pod_cidr}"

  aws ec2 create-route \
    --route-table-id "${ROUTE_TABLE_ID}" \
    --destination-cidr-block "${pod_cidr}" \
    --instance-id "${instance_id}"
done
```
Nos aparecerá algo como esto en consola:

```
10.0.1.20 10.200.0.0/24
{
    "Return": true
}
10.0.1.21 10.200.1.0/24
{
    "Return": true
}
10.0.1.22 10.200.2.0/24
{
    "Return": true
}
```

## Validando las rutas<a id="15_2"></a>

Ahora tan sólo queda validar las rutas en cada worker-node:

```
aws ec2 describe-route-tables \
  --route-table-ids "${ROUTE_TABLE_ID}" \
  --query 'RouteTables[].Routes'
```

Lo cual nos devolverá algo como esto:

```
[
    [
        {
            "DestinationCidrBlock": "10.200.0.0/24",
            "InstanceId": "i-0879fa49c49be1a3e",
            "InstanceOwnerId": "107995894928",
            "NetworkInterfaceId": "eni-0612e82f1247c6282",
            "Origin": "CreateRoute",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "10.200.1.0/24",
            "InstanceId": "i-0db245a70483daa43",
            "InstanceOwnerId": "107995894928",
            "NetworkInterfaceId": "eni-0db39a19f4f3970f8",
            "Origin": "CreateRoute",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "10.200.2.0/24",
            "InstanceId": "i-0b93625175de8ee43",
            "InstanceOwnerId": "107995894928",
            "NetworkInterfaceId": "eni-0cc95f34f747734d3",
            "Origin": "CreateRoute",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "10.0.0.0/16",
            "GatewayId": "local",
            "Origin": "CreateRouteTable",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "0.0.0.0/0",
            "GatewayId": "igw-00d618a99e45fa508",
            "Origin": "CreateRoute",
            "State": "active"
        }
    ]
]
```

# Configurando el DNS add-on<a id="16"></a>

Vamos a configurar el [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/). Esta herramienta sirve para asignar una DNS a cada una de las IPs que se corresponden con un pod o un servicio, lo cual es de gran ayuda en un contexto con muchos pods y servicios corriendo a la vez.

## Creando el DNS add-on<a id="16_1"></a>

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml

```

En consola aparecerán las herramientas que se van instalando:

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

Como podemos ver, DNS add-on está basado en [CoreDNS](https://coredns.io/). Y éste a su vez se crea en forma de pods en nuestro cluster. Podemos verlo si listamos los pods del namespace kube-system:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```
La salida será algo como esto:

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-hh7r2   1/1     Running   0          10s
coredns-8494f9c688-zqrj2   1/1     Running   0          10s

```

## Verificación<a id="16_2"></a>

Vamos a verificarlo creando un pod llamado busybox con una imagen de busybox.

```
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

Comprobamos que se ha creado correctamente:

```
kubectl get pods -l run=busybox
```
La salida:

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Vamos a buscar el pod que hemos mediante consultando los registros DNS para ver que nuestro DNS add-on funciona correctamente.

Primero obtenemos el nombre del pod:

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Y buscamos ejecutando el comando nslookup:

```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

Lo cual debería devolver algo como

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local
Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

# Probando nuestro cluster<a id="17"></a>

Por fin tenemos el cluster funcionando. Ahora vamos a ejecutar algunas tareas para comprobar que funciona correctamente.

## Encriptación de los datos<a id="17_1"></a>

Vamos a generar un secreto (https://kubernetes.io/docs/concepts/configuration/secret/) aleatorio:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Y ahora vamos a conectarnos al cluster para ver el secreto que hemos creado con el nombre kubernetes-the-hard-way y que se ha almacenado en etcd (y por ende, utilizamos el comando etcctl get con la version API=3): 

```
external_ip=$(aws ec2 describe-instances --filters \
  "Name=tag:Name,Values=controller-0" \
  "Name=instance-state-name,Values=running" \
  --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip} \
 "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"

```

La consola mostrará algo como :

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 97 d1 2c cd 89 0d 08  |:v1:key1:..,....|
00000050  29 3c 7d 19 41 cb ea d7  3d 50 45 88 82 a3 1f 11  |)<}.A...=PE.....|
00000060  26 cb 43 2e c8 cf 73 7d  34 7e b1 7f 9f 71 d2 51  |&.C...s}4~...q.Q|
00000070  45 05 16 e9 07 d4 62 af  f8 2e 6d 4a cf c8 e8 75  |E.....b...mJ...u|
00000080  6b 75 1e b7 64 db 7d 7f  fd f3 96 62 e2 a7 ce 22  |ku..d.}....b..."|
00000090  2b 2a 82 01 c3 f5 83 ae  12 8b d5 1d 2e e6 a9 90  |+*..............|
000000a0  bd f0 23 6c 0c 55 e2 52  18 78 fe bf 6d 76 ea 98  |..#l.U.R.x..mv..|
000000b0  fc 2c 17 36 e3 40 87 15  25 13 be d6 04 88 68 5b  |.,.6.@..%.....h[|
000000c0  a4 16 81 f6 8e 3b 10 46  cb 2c ba 21 35 0c 5b 49  |.....;.F.,.!5.[I|
000000d0  e5 27 20 4c b3 8e 6b d0  91 c2 28 f1 cc fa 6a 1b  |.' L..k...(...j.|
000000e0  31 19 74 e7 a5 66 6a 99  1c 84 c7 e0 b0 fc 32 86  |1.t..fj.......2.|
000000f0  f3 29 5a a4 1c d5 a4 e3  63 26 90 95 1e 27 d0 14  |.)Z.....c&...'..|
00000100  94 f0 ac 1a cd 0d b9 4b  ae 32 02 a0 f8 b7 3f 0b  |.......K.2....?.|
00000110  6f ad 1f 4d 15 8a d6 68  95 63 cf 7d 04 9a 52 71  |o..M...h.c.}..Rq|
00000120  75 ff 87 6b c5 42 e1 72  27 b5 e9 1a fe e8 c0 3f  |u..k.B.r'......?|
00000130  d9 04 5e eb 5d 43 0d 90  ce fa 04 a8 4a b0 aa 01  |..^.]C......J...|
00000140  cf 6d 5b 80 70 5b 99 3c  d6 5c c0 dc d1 f5 52 4a  |.m[.p[.<.\....RJ|
00000150  2c 2d 28 5a 63 57 8e 4f  df 0a                    |,-(ZcW.O..|
0000015a
```

Vemos que la llave viene precedida de `k8s:enc:aescbc:v1:key1 ` lo cual nos indicata que se ha utilizado el cifrado  [aescbc](https://es.wikipedia.org/wiki/Modos_de_operaci%C3%B3n_de_una_unidad_de_cifrado_por_bloques)  utilizando la  llave de encriptación key1 que hemos creado anteriormente durante la [Encriptación de los datos en los Controller-Nodes](#8_1).

## Creando deployments<a id="17_2"></a>

Ahora vamos a crear un deployment utilizando la imagen de `nginx`.

```
kubectl create deployment nginx --image=nginx
```

Podemos ver si se han creado los pods correspondientes:

```
kubectl get pods -l app=nginx
```

```

NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-kpn5m   1/1     Running   0          10s
```

## Redireccionamiento de puertos<a id="17_3"></a>

Vamos a crear un redireccionamiento de puertos (un Port Forwarding) para acceder nuestro pod remotamente. Primero recuperamos el nombre de nuestro pod:

```

POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Y aplicamos un redireccionamiento de puertos. En este caso mapeamos el puerto 8080 de nuestra máquina local al puerto 80 del pod:

```
kubectl port-forward $POD_NAME 8080:80
```
Nos aparecerá esto en consola:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Abrimos una nueva terminal e intentamos conectarnos:

```
curl --head http://127.0.0.1:8080
```

Si todo ha salido bien, nos devolverá la siguiente cabecera:

```
HTTP/1.1 200 OK
Server: nginx/1.21.1
Date: Sat, 07 Aug 2021 21:08:34 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 06 Jul 2021 14:59:17 GMT
Connection: keep-alive
ETag: "60e46fc5-264"
Accept-Ranges: bytes
```

Ahora volvemos a la terminal desde donde habíamos aplicado el redireccionamiento y lo paramos con `^C `.

## Logs<a id="17_4"></a>

Vamos a comprobar que los logs se crean y se almacenan de forma correcta.

```
kubectl logs $POD_NAME
```

Nos debería salir algo parecido a  lo siguiente:

```
127.0.0.1 - - [07/Aug/2021:21:08:34 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.64.1" "-"
```

## Exec<a id="17_5"></a>

Lo siguiente es comprobar si podemos ejecutar comandos en el pod. En este caso vamos a imprimir la versión de nginx mediante el comando `nginx -v`, pero en este [enlace se pueden ver más comando por si nos apetece seguir indagando](https://kubernetes.io/docs/reference/kubectl/cheatsheet/).

```
kubectl exec -ti $POD_NAME -- nginx -v
```

Aparecerá en consola lo siguiente:

```
nginx version: nginx/1.21.1
```

## Services<a id="17_6"></a>

Por último vamos a exponer nuestro deployment mediante un [servicio](https://kubernetes.io/es/docs/concepts/services-networking/service/), en este caso del tipo NodePort.

```
kubectl expose deployment nginx --port 80 --type NodePort
```

Vamos a guardar el puerto asignado al NodePort:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Y añadimos una regla a nuestro grupo de seguridad (es decir, un firewall) para poder acceder al NodePort y así poder conectar con el deployment:

```
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port ${NODE_PORT} \
  --cidr 0.0.0.0/0
```

Y ahora vamos a intentar conectarnos. Primero obtenemos el nombre del nodo en el que se ha creado el deployment:

```
INSTANCE_NAME=$(kubectl get pod $POD_NAME --output=jsonpath='{.spec.nodeName}')
```

Y su dirección IP:

```
EXTERNAL_IP=$(aws ec2 describe-instances --filters \
    "Name=instance-state-name,Values=running" \
    "Name=network-interface.private-dns-name,Values=${INSTANCE_NAME}.*.internal*" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
```

Ya sólo nos queda hacer un HTTP request a la IP del nodo y al puerto que utilizamos como NodePort:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

Y nos aparecerá algo como:

```
HTTP/1.1 200 OK
Server: nginx/1.21.1
Date: Sat, 07 Aug 2021 21:16:44 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 06 Jul 2021 14:59:17 GMT
Connection: keep-alive
ETag: "60e46fc5-264"
Accept-Ranges: bytes
```

# Limpieza<a id="18"></a>

NOTA: Recomiendo hacer la limpieza y la eliminación de recursos de forma manual a través de la consola de aws para evitar gastos inesperados. Los comandos que aparecen a continuación no funcionaron al 100% en mi caso, ya que como expliqué anteriormente, tardé más de un día en completar el ejercicio y algunas variables que aquí se utilizan se habían eliminado así que recurrí al método manual para asegurarme que todo había sido eliminado.

Si has completado el ejercicio en unas horas los comandos funcionaran, pero sigo recomendando echar un vistazo a la consola de AWS.

## Eliminar instancias<a id="18_1"></a>

Eliminamos las instancias con:

```
echo "Issuing shutdown to worker nodes.. " && \
aws ec2 terminate-instances \
  --instance-ids \
    $(aws ec2 describe-instances --filters \
      "Name=tag:Name,Values=worker-0,worker-1,worker-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Waiting for worker nodes to finish terminating.. " && \
aws ec2 wait instance-terminated \
  --instance-ids \
    $(aws ec2 describe-instances \
      --filter "Name=tag:Name,Values=worker-0,worker-1,worker-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Issuing shutdown to master nodes.. " && \
aws ec2 terminate-instances \
  --instance-ids \
    $(aws ec2 describe-instances --filter \
      "Name=tag:Name,Values=controller-0,controller-1,controller-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Waiting for master nodes to finish terminating.. " && \
aws ec2 wait instance-terminated \
  --instance-ids \
    $(aws ec2 describe-instances \
      --filter "Name=tag:Name,Values=controller-0,controller-1,controller-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')

aws ec2 delete-key-pair --key-name kubernetes
```

## Eliminar el balanceador de carga, el gateway y los demás recursos de la red<a id="18_2"></a>

Y ahora el resto de recursos:

```
aws elbv2 delete-load-balancer --load-balancer-arn "${LOAD_BALANCER_ARN}"
aws elbv2 delete-target-group --target-group-arn "${TARGET_GROUP_ARN}"
aws ec2 delete-security-group --group-id "${SECURITY_GROUP_ID}"
ROUTE_TABLE_ASSOCIATION_ID="$(aws ec2 describe-route-tables \
  --route-table-ids "${ROUTE_TABLE_ID}" \
  --output text --query 'RouteTables[].Associations[].RouteTableAssociationId')"
aws ec2 disassociate-route-table --association-id "${ROUTE_TABLE_ASSOCIATION_ID}"
aws ec2 delete-route-table --route-table-id "${ROUTE_TABLE_ID}"
echo "Waiting a minute for all public address(es) to be unmapped.. " && sleep 60

aws ec2 detach-internet-gateway \
  --internet-gateway-id "${INTERNET_GATEWAY_ID}" \
  --vpc-id "${VPC_ID}"
aws ec2 delete-internet-gateway --internet-gateway-id "${INTERNET_GATEWAY_ID}"
aws ec2 delete-subnet --subnet-id "${SUBNET_ID}"
aws ec2 delete-vpc --vpc-id "${VPC_ID}"
```
