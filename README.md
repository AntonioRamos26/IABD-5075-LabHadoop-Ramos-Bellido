# IABD-5075-LabHadoop-Ramos-Bellido

**Despliegue de un clúster multi-nodo Apache Hadoop con Docker, auditoría física en HDFS y simulación de tolerancia a fallos**

Laboratorio (AEE) · Módulo 5074/5075 · Máster en Inteligencia Artificial y Big Data

**Autor:** Antonio Ramos Bellido

---

## 1. Descripción

Este repositorio contiene la infraestructura como código (IaC), las evidencias y el informe técnico de un laboratorio en el que se despliega un clúster Apache Hadoop con **Docker Compose**, replicando en una sola máquina la separación entre nodos maestros y trabajadores. Sobre él se realiza:

1. Despliegue y auditoría de salud del clúster (HDFS + YARN).
2. Ingesta de un fichero de 150 MiB y auditoría física de bloques con `hdfs fsck`.
3. Simulación de la caída de un DataNode y verificación de la tolerancia a fallos.
4. Ejecución de un trabajo MapReduce (*WordCount*) sobre YARN.

## 2. Arquitectura desplegada

Seis contenedores conectados mediante una red puente aislada (`hadoop-net`), imagen `apache/hadoop:3`.

```
                      Equipo anfitrión (Docker)  ·  red puente "hadoop-net"
 ┌────────────────────────────────────────────────────────────────────────┐
 │  NODOS MAESTROS                       NODOS TRABAJADORES               │
 │  ┌──────────────┐ ┌────────────────┐  ┌───────────┐ ┌───────────┐      │
 │  │  namenode    │ │ resourcemanager│  │ datanode1 │ │ datanode2 │      │
 │  │  metadatos   │ │ planificador   │  └───────────┘ └───────────┘      │
 │  │  HDFS :9870  │ │ YARN :8088     │  ┌───────────┐ ┌─────────────┐    │
 │  └──────────────┘ └────────────────┘  │ datanode3 │ │ nodemanager │    │
 │                                       └───────────┘ └─────────────┘    │
 │                                         bloques HDFS   contenedores    │
 │                                                        YARN            │
 └────────────────────────────────────────────────────────────────────────┘
```

| Servicio          | Rol                                                     | Puerto publicado |
|-------------------|---------------------------------------------------------|------------------|
| `namenode`        | Metadatos de HDFS y mapa fichero → bloques → nodos      | 9870 (web)       |
| `datanode1`       | Almacena bloques HDFS y envía *heartbeats*              | —                |
| `datanode2`       | Almacena bloques HDFS y envía *heartbeats*              | —                |
| `datanode3`       | Almacena bloques HDFS y envía *heartbeats*              | —                |
| `resourcemanager` | Planificador y gestor de recursos de YARN               | 8088 (web)       |
| `nodemanager`     | Ejecuta los contenedores de las tareas MapReduce        | —                |

### Parámetros principales (`docker/config`)

| Parámetro                         | Valor                  | Significado                              |
|-----------------------------------|------------------------|------------------------------------------|
| `dfs.blocksize`                   | `67108864` (64 MiB)    | Tamaño de bloque HDFS                    |
| `dfs.replication`                 | `2`                    | Factor de replicación R = 2              |
| `dfs.permissions.enabled`         | `false`                | Sin comprobación de permisos (laboratorio) |
| `yarn.nodemanager.resource.memory-mb` | `2048`             | Memoria que ofrece el NodeManager        |
| `yarn.nodemanager.resource.cpu-vcores`| `2`                | vCores que ofrece el NodeManager         |

Además se añadieron ajustes de memoria para MapReduce (`yarn.app.mapreduce.am.resource.mb`, `mapreduce.map.memory.mb`, `mapreduce.reduce.memory.mb`, sus `java.opts` y `yarn.scheduler.minimum-allocation-mb=256`), necesarios para que un NodeManager de 2048 MB pueda ejecutar el ApplicationMaster y las tareas map a la vez. Se detallan en el informe.

## 3. Requisitos

- Docker Desktop (o Docker Engine + Docker Compose v2) en ejecución.
- Git y una cuenta de GitHub.
- Al menos **6 GB de RAM libres**.
- Navegador web y terminal (PowerShell, Terminal de Windows o Bash).

## 4. Instrucciones de arranque

```bash
git clone https://github.com/AntonioRamos26/IABD-5075-LabHadoop-Ramos-Bellido.git
cd IABD-5075-LabHadoop-Ramos-Bellido/docker

docker compose up -d        # levanta los 6 servicios
docker compose ps           # todos deben estar "Up"
# (en instalaciones antiguas de Docker: docker-compose en lugar de docker compose)
```

Interfaces web:

- NameNode (HDFS): <http://localhost:9870> → pestaña **Datanodes**
- ResourceManager (YARN): <http://localhost:8088>

Verificación desde el contenedor maestro:

```bash
docker exec -it namenode bash
hdfs dfsadmin -report       # debe indicar "Live datanodes (3)"
```

Parada y limpieza (elimina los datos del clúster):

```bash
docker compose down -v
```

## 5. Resumen de los experimentos

| Fase | Contenido                                                                 | Evidencia                                   |
|------|---------------------------------------------------------------------------|---------------------------------------------|
| 2    | Despliegue y registro de los 3 DataNodes                                  | `evidencias/01_namenode_ui_9870.png`        |
| 3    | Fichero de 150 MiB → 3 bloques (64 + 64 + 22 MiB), R = 2 → 6 réplicas      | `evidencias/02_hdfs_fsck_bloques.png`       |
| 4    | Parada de `datanode2` y observación del silencio de *heartbeats* (*Last contact* creciente, estado *Stale*) | `evidencias/03_simulacion_fallo_datanode2.png` |
| 5    | *WordCount* sobre YARN (FINISHED / SUCCEEDED)                              | `evidencias/04_yarn_wordcount_8088.png`     |

El análisis completo y las respuestas a las cuatro cuestiones técnicas están en el informe:
[`informe/Ramos_Bellido_InformeLabHadoop.pdf`](informe/Ramos_Bellido_InformeLabHadoop.pdf).

## 6. Estructura del repositorio

```
IABD-5075-LabHadoop-Ramos-Bellido/
├── .gitignore
├── README.md
├── docker/
│   ├── config
│   └── docker-compose.yml
├── evidencias/
│   ├── 01_namenode_ui_9870.png
│   ├── 02_hdfs_fsck_bloques.png
│   ├── 03_simulacion_fallo_datanode2.png
│   └── 04_yarn_wordcount_8088.png
└── informe/
    └── Ramos_Bellido_InformeLabHadoop.pdf
```

El fichero de prueba `telemetria_bruta.dat` (150 MiB) se genera dentro del contenedor y **no** se versiona (ver `.gitignore`).

## 7. Autoría

Antonio Ramos Bellido · Máster en Inteligencia Artificial y Big Data · Noviembre de 2026
