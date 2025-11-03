# Actividades prácticas y de análisis
## 1.Verificación de replicación

**En el cliente**

```bash
docker exec -it client bash
echo 'Archivo 1' > /mnt/dfs/archivo1.txt
echo 'Archivo 2' > /mnt/dfs/archivo2.txt
echo 'Archivo 3' > /mnt/dfs/archivo3.txt
exit
```

**En el nodo 1**
```bash
docker exec -it node1 bash
cat /data/brick1/archivo1.txt
cat /data/brick1/archivo2.txt
cat /data/brick1/archivo3.txt
exit
```

**En el nodo 2**
```bash
docker exec -it node2 bash
cat /data/brick1/archivo1.txt
cat /data/brick1/archivo2.txt
cat /data/brick1/archivo3.txt
exit
```

**Resultados:**
```bash
PS C:\Users\Emma\Desktop\dfs-lab> docker exec -it node1 bash
[root@node1 /]# cat /data/brick1/archivo1.txt
Archivo 1
[root@node1 /]# cat /data/brick1/archivo2.txt
Archivo 2
[root@node1 /]# cat /data/brick1/archivo3.txt
Archivo 3
[root@node1 /]# exit
exit
PS C:\Users\Emma\Desktop\dfs-lab> docker exec -it node2 bash
[root@node2 /]# cat /data/brick1/archivo1.txt
Archivo 1
[root@node2 /]# cat /data/brick1/archivo2.txt
Archivo 2
[root@node2 /]# cat /data/brick1/archivo3.txt
Archivo 3
[root@node2 /]# exit
exit
PS C:\Users\Emma\Desktop\dfs-lab>
```

## 2.Prueba de escritura concurrente
Usaremos dos terminales para simular concurrencia, los clientes seran cliente y el nodo 1

### 1.Preparamos ambos "clientes"
**Terminal 1**
```bash
docker exec -it client bash

```

**Terminal 2**
```bash
docker exec -it node1 bash
mkdir /mnt/dfs
mount -t glusterfs node1:/datavolume /mnt/dfs
```

### 2.Escritura Concurrente
Ejecutamos al mismo tiempo un comando de escritura en cada terminal, intentando escribir al mismo el archivo *concurrente.txt*.

**Terminal 1**
```bash
echo "ESCRITURA 1: Desde Cliente 1 (cliente)" > /mnt/dfs/concurrente.txt

```

**Terminal 2**
```bash
echo "ESCRITURA 2: Desde Cliente 2 (nodo 1)" > /mnt/dfs/concurrente.txt
```

### 2.Observemos el resultado
```bash
cat /mnt/dfs/concurrente.txt
```

**Resultado**
```bash
ESCRITURA 2: Desde Cliente 2 (nodo 1)
```

## 3.Medición de rendimiento
Abrimos el cliente
```bash
docker exec -it client bash
```

### Prueba de escritura secuencial
Mide la velocidad a la que el sistema puede escribir un archivo grande en el volumen.
| Parámetro | Significado |
| :--- | :--- |
| `if=/dev/zero` | Fuente de datos (ceros, lo que elimina el factor de rendimiento del disco de origen). |
| `of=/mnt/dfs/testfile_write` | Destino (el volumen distribuido de GlusterFS). |
| `bs=1M` | Tamaño del bloque de lectura/escritura (1 Megabyte). |
| `count=1024` | Número de bloques a escribir (Total: 1024 MB=1 GB). |
| `conv=fdatasync` | Fuerza a que los datos se escriban realmente en el disco, no solo en la caché, dando una medición más real del rendimiento del DFS. |

```bash
docker exec -it client bash
dd if=/dev/zero of=/mnt/dfs/testfile_write bs=1M count=1024 conv=fdatasync
```
**Resultado**
```bash
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 8.37944 s, 128 MB/s
```
La velocidad de escritura fue de de 128 MB/s

### Prueba de lectura secuencial
Mide la velocidad a la que el sistema puede leer el archivo recien creado.
| Parámetro | Significado |
| :--- | :--- |
| `if=/mnt/dfs/testfile_write` | Archivo de origen (el archivo de 1 GB que escribiste en el DFS). |
| `of=/dev/null` | "Destino para que el rendimiento solo dependa de la velocidad de lectura del disco. |

```bash
dd if=/mnt/dfs/testfile_write of=/dev/null bs=1M
```
**Resultado**
```bash
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 0.268309 s, 4.0 GB/s
```
La velocidad de lectura fue de de 4.0 GB/s

## 4.Extensión del clúster
**1. Agregamos el nodo 3  *docker-compose.yml***
**2. Levantamos el docker**
```bash
docker-compose up -d
```
**3. Crear el segundo directorio/brick en el volumen de node1** 
Esto es porque anteriormente usamos replica 2, asi que necesitamos 4 bricks
```bash
docker exec -it node1 bash
mkdir -p /data/brick2
```
**4. Añadimos el nodo 3 al cluster**
```bash
gluster peer probe node3
gluster peer status
```
**5. Añade el nuevo brick al volumen datavolume**
```bash
gluster volume add-brick datavolume node3:/data/brick1 node1:/data/brick2 force
gluster volume info
```
**6. Rebalanceamos**
```bash
gluster volume rebalance datavolume start
gluster volume rebalance datavolume status
exit
```