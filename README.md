1. Inicializar el entorno
Iniciar los contenedores:

docker-compose up -d

Verificar que los tres estén activos:

docker ps

Deberías ver algo como:

CONTAINER ID NAME STATUS PORTS
a1b23c45d6e7 node1 Up
b2c34d45e6f7 node2 Up
c3d45e56f7g8 client Up

2. Configurar GlusterFS en los nodos
Ingresar al primer nodo:

docker exec -it node1 bash

Agregar el segundo nodo al clúster:

gluster peer probe node2
gluster peer status

3. Crear el volumen distribuido y replicado
Desde node1 : 

gluster volume create datavolume replica 2 transport tcp node1:/data/brick1 node2:/data/brick1 force
gluster volume start datavolume
gluster volume info
exit

4. Montar el sistema de archivos en el cliente
Acceder al contenedor cliente:

docker exec -it client bash

*Actualizar los Archivos de Repositorio*
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' CentOS-*
yum clean all


Instalar herramientas necesarias:

yum install -y glusterfs glusterfs-fuse

Montar el volumen:

mkdir /mnt/dfs
mount -t glusterfs node1:/datavolume /mnt/dfs

Verificar el montaje:

df -hT | grep gluster
exit

5. Prueba de replicación
En el cliente:

echo 'Hola DFS Distribuido!' > /mnt/dfs/prueba.txt
exit

En el nodo 2:

docker exec -it node2 bash
cat /data/brick1/prueba.txt

*Si aparece el texto (Hola DFS Distribuido!), la replicación funciona correctamente.*
exit

6. Simulación de tolerancia a fallos
Apagar el nodo 1:

docker stop node1

En el cliente:

cat /mnt/dfs/prueba.txt

Deberías poder seguir accediendo al archivo gracias a la réplica en node2.
Volver a encender el nodo:

docker start node1
