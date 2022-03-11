---
layout: post
title: Configurar instancias de EC2 mediante AWS CLI
description: Una sencilla guía para configurar y lanzar instancias de EC2 mediante AWS CLI
tags: [ AWS , EC2, DevOps ]
---

# Indice
1. [Introducción](#introduction)

2. [Configuración de AWS CLI](#configuracion)

3. [Networking](#networking)
    1. [Virtual Private Cloud (VPC)](#vpc)
    2. [Subnet](#subnet)
    3. [Internet Gateway y tabla de enrutamiento](#gateway)
    4. [Security Group](#security)

4. [Crear una instancia de ECS](#instancia) 


# Introducción <a id="introduction"></a>

En el post de hoy voy a describir los pasos a seguir para poder conectarse a AWS y trabajar mediante la terminal instalando la [CLI de AWS](https://aws.amazon.com/es/cli/).

IMPORTANTE: El siguiente ejercicio puede conllevar algunos costes si no estás dentro de la capa gratuíta de AWS. Asegúrate de borrar todas las instancias una vez termines.



# Configuración de AWS CLI <a id="configuracion"></a>

El primer paso es instalar la [CLI en la terminal](https://github.com/aws/aws-cli). Tal y como pone en la documentación, la manera más segura es usar pip en un virtualenv:
```
python -m pip install awscli
```

Una vez que la instalación se ha ejecutado con éxito, configuro la zona horaria  (eu-central-1 en mi caso). Puedes ver más información sobre zonas horarias [aquí](https://docs.aws.amazon.com/es_es/es_es/redshift/latest/dg/concurrency-scaling-regions.html)

```
AWS_REGION=eu-central-1 
aws configure set default.region $AWS_REGION
```

También es necesario configurar el acceso creando un named profile. Para generar las claves hay que ir a la consola de AWS (mediante el navegador) y dirigirse a la sección `Identity and Access Management (IAM)`. Una vez ahí, bajo `Administración del acceso -> Grupos` crearemos un grupo con los permisos necesarios y luego volveremos a Usuarios donde crearemos un nuevo usuario (todo esto es si no tenemos ya uno). En `Usuarios -> Credenciales de seguridad` crearemos una clave de acceso que serán las claves que necesitaremos para conectarnos a AWS mediante la terminal. Una vez tenemos las claves, podemos volver a la terminal y hacer: 

```
aws config
```

Nos pedirá la Access key ID y el Secret Access key. Una vez completado todo esto podemos empezar a trabajar.  


# Networking<a id="networking"></a>

## Virtual Private Cloud (VPC)<a id="vpc"></a>

Una VPC (Virtual Private Cloud ) es una porción aislada de la nube de AWS dentro de una región determinada. Dentro de una VPC puedes crear subnets públicas o bien privadas, y meter recursos en ellas. 

Cada VPC tiene un bloque de direcciones IP. Aquí entra el concepto de CIDR (Classless Inter-Domain Routing, también llamado superred) que definimos con el parámetro `--cidr-bloc` (en este caso 10.0.0.0/16).

Primeramente crearemos la VPC de la siguiente manera:

´´´
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output text --query 'Vpc.VpcId')

aws ec2 create-tags --resources ${VPC_ID} --tags Key=Name,Value=mi-vpc
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support '{"Value": true}'
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames '{"Value": true}'
´´´

## Subnet<a id="subnet"></a>

Como he puesto arriba, cada VPC tiene una o varias subnets con un bloque de direcciones IP. La creamos de la siguiente forma. Lógicamente el bloque de IPs de la subnet es menor que el de la VPC.

´´´
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.0.1.0/24 \
  --output text --query 'Subnet.SubnetId')

aws ec2 create-tags --resources ${SUBNET_ID} --tags Key=Name,Value=mi-subnet
´´´

## Internet Gateway y tabla de enrutamiento<a id="gateway"></a>

Lo siguiente que haremos será crear una internet gateway y una tabla de enrutamiento. Esta tabla define qué tráfico se mueve dentro de nuestra VPC y qué tráfico se dirigirá a internet mediante la Internet Gateway.

Internet Gateway:

```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId')

aws ec2 create-tags --resources ${INTERNET_GATEWAY_ID} --tags Key=Name,Value=mi-gateway
aws ec2 attach-internet-gateway --internet-gateway-id ${INTERNET_GATEWAY_ID} --vpc-id ${VPC_ID}
```

Tabla de enrutamiento:

```
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')

aws ec2 create-tags --resources ${ROUTE_TABLE_ID} --tags Key=Name,Value=kubernetes
aws ec2 associate-route-table --route-table-id ${ROUTE_TABLE_ID} --subnet-id ${SUBNET_ID}
aws ec2 create-route --route-table-id ${ROUTE_TABLE_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${INTERNET_GATEWAY_ID}
```

## Security Group<a id="security"></a>

Ahora crearemos un grupo de seguridad , que no es otra cosa que un firewall que aceptará las conexiones entrantes mediante ciertos puertos:

```
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name seguridad \
  --description "mi-grupo-de-seguridad" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

aws ec2 create-tags --resources ${SECURITY_GROUP_ID} --tags Key=Name,Value=mi-grupo-de-seguridad

//autorizamos conexiones ssh
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
//autorizamos conexiones https
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 443 --cidr 0.0.0.0/0
//autorizamos conexiones http
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 80 --cidr 0.0.0.0/0
```

# Crear una instancia de ECS<a id="instancia"></a>

Una vez configurado todo lo anterior, el último paso es crear un par llave-valor y cambiar los permisos para poder ejecutarlo:

```
aws ec2 create-key-pair --key-name miLlaveValor
chmod 400 miLlaveValor.pem
```

Con todo lo anterior me dispongo a crear una instancia de EC2 (en este caso una imagen de Ubuntu Server 20.04 LTS (HVM), SSD Volume Type con un tipo de instancia t2.micro):

```
aws ec2 run-instances --image-id ami-0d527b8c289b4af7f --count 1 --instance-type t2.micro --key- name <miLlaveValor> --security-group-ids <id-del-security-group> --subnet-id <id-de-la-subnet> -- associate-public-ip-address
````

Una vez hayamos completado el ejercicio, no hay que olvidar borrar todas las instancias si no queremos tener gastos no deseados.

