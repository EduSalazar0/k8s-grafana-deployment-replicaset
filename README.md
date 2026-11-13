# Grafana en Kubernetes con Deployment y ReplicaSet

Tarea individual. Se despliega **Grafana** (plataforma de visualización y monitoreo) en un clúster local de **minikube**, aplicando los conceptos de **Deployment** y **ReplicaSet**, y se expone fuera del clúster con un **Service NodePort**.

| Elemento | Valor |
|---|---|
| Clúster | minikube, perfil `taller` (driver Docker, Kubernetes v1.37.0) |
| Namespace | `grafana-lab` |
| Imagen | `grafana/grafana:11.6.0` (actualización de prueba a `12.0.0`) |
| Réplicas | 3 |
| Exposición | Service `grafana-svc`, NodePort `30300` |
| Credenciales de laboratorio | usuario `admin` / clave definida en [`01-grafana-secret.yml`](grafana/01-grafana-secret.yml) |

## Estructura del repositorio

```
.
├── pod_gnix/
│   └── first_pod_nginx.yml          # Pod de nginx visto en clase (punto de partida)
├── grafana/
│   ├── 00-namespace.yml             # Namespace grafana-lab
│   ├── 01-grafana-secret.yml        # Usuario y clave admin (Secret)
│   ├── 02-grafana-configmap.yml     # Datasource + dashboard provisionados (ConfigMap)
│   ├── 03-grafana-deployment.yml    # Deployment con 3 réplicas y RollingUpdate
│   ├── 04-grafana-service.yml       # Service NodePort 30300
│   └── 05-replicaset-demo.yml       # ReplicaSet "solo", para comparar con el Deployment
├── evidencias/                      # Salidas reales de kubectl de cada prueba
│   └── capturas/                    # Capturas de pantalla de cada paso (terminal y Grafana)
├── Informe_Tarea_K8s_Grafana.docx   # Informe técnico (Word)
└── Informe_Tarea_K8s_Grafana.pdf    # Informe técnico (PDF)
```

El informe técnico completo, con capturas y análisis de cada paso, está en [Informe_Tarea_K8s_Grafana.pdf](Informe_Tarea_K8s_Grafana.pdf).

![Dashboard de Grafana desplegado en Kubernetes](evidencias/capturas/04-grafana-dashboard.png)

## Conceptos

### Del Pod al Deployment
En clase se creó un **Pod** suelto (`myfirstpod`). Si ese Pod muere o se borra, nadie lo vuelve a crear. Para una plataforma real se necesitan objetos que **mantengan** los Pods:

```
Deployment (grafana)
   └── ReplicaSet (grafana-68655b84c8)   ← lo crea el Deployment
          ├── Pod grafana-68655b84c8-xxxxx
          ├── Pod grafana-68655b84c8-yyyyy
          └── Pod grafana-68655b84c8-zzzzz
Service NodePort (grafana-svc :30300) → balancea entre los Pods con app=grafana
```

### ReplicaSet
- Garantiza que **siempre** exista el número de Pods indicado en `replicas`.
- Identifica sus Pods por **etiquetas** (`selector.matchLabels`).
- Si un Pod se borra o falla, crea otro (**autorreparación**). Si sobran, elimina los que sobran.
- **Limitación:** si cambias la imagen en su plantilla, los Pods existentes **no** se actualizan y no hay historial ni rollback.

### Deployment
- Es el objeto que normalmente se usa. **No crea Pods directamente:** crea y administra ReplicaSets.
- Cada cambio en la plantilla del Pod (por ejemplo, la versión de la imagen) genera un **ReplicaSet nuevo**, y el Deployment mueve los Pods del viejo al nuevo poco a poco (**RollingUpdate**).
- Guarda el **historial de revisiones** (`revisionHistoryLimit`) y permite volver atrás con `kubectl rollout undo`.
- Con `maxSurge: 1` y `maxUnavailable: 0` la actualización se hace **sin caída**: siempre hay 3 Pods disponibles.

### Otros objetos usados
- **Namespace:** aísla los recursos de la tarea.
- **Secret:** credenciales del administrador, inyectadas como variables de entorno.
- **ConfigMap:** configuración declarativa (datasource y dashboard), igual en todas las réplicas.
- **Probes:** `readinessProbe` (solo recibe tráfico cuando está listo) y `livenessProbe` (se reinicia si se cuelga), ambas contra `/api/health`.
- **Service NodePort:** expone Grafana en el puerto `30300` del nodo y reparte el tráfico entre las réplicas. Usa `sessionAffinity: ClientIP` porque cada réplica guarda su propia sesión de login.

## Cómo ejecutar el proyecto

### 1. Requisitos

| Herramienta | Versión probada | Instalación |
|---|---|---|
| Docker Desktop (encendido) | 29.x | https://docs.docker.com/get-docker/ |
| minikube | v1.39.0 | https://minikube.sigs.k8s.io/docs/start/ |
| kubectl | compatible con Kubernetes 1.37 | https://kubernetes.io/docs/tasks/tools/ |
| Git | cualquiera | https://git-scm.com/ |

Se necesitan unos **2 GB de RAM libres** para el clúster y conexión a internet la primera vez (descarga de la imagen `grafana/grafana`, unos 200 MB).

### 2. Clonar el repositorio

```bash
git clone https://github.com/EduSalazar0/k8s-grafana-deployment-replicaset.git
```

```bash
cd k8s-grafana-deployment-replicaset
```

### 3. Iniciar el clúster

```bash
minikube start -p taller --driver=docker
```

El perfil `taller` se crea solo si no existe. Se puede usar cualquier otro nombre, o el perfil por defecto con `minikube start`. En ese caso hay que quitar `-p taller` de los comandos siguientes.

### 4. Desplegar Grafana

```bash
kubectl apply -f grafana/00-namespace.yml -f grafana/01-grafana-secret.yml -f grafana/02-grafana-configmap.yml -f grafana/03-grafana-deployment.yml -f grafana/04-grafana-service.yml
```

```bash
kubectl -n grafana-lab rollout status deploy/grafana
```

```bash
kubectl -n grafana-lab get deploy,rs,pods,svc -o wide
```

Resultado esperado: `deployment.apps/grafana 3/3`, un ReplicaSet `grafana-68655b84c8` con 3 réplicas listas, 3 Pods `Running` y `service/grafana-svc NodePort 80:30300/TCP`. La primera vez puede tardar 1–2 minutos mientras descarga la imagen.

### 5. Abrir Grafana

```bash
minikube service grafana-svc -n grafana-lab -p taller
```

- **Windows / macOS (driver Docker):** minikube abre un túnel en `http://127.0.0.1:<puerto>` y lanza el navegador. **La terminal debe quedar abierta** mientras se usa Grafana. Los demás comandos se ejecutan en otra terminal.
- **Linux:** también se puede entrar directo a `http://<minikube ip -p taller>:30300`.

**Credenciales:** usuario `admin`, contraseña `Grafana-Lab-2026` (definidas en [`01-grafana-secret.yml`](grafana/01-grafana-secret.yml)).

Al iniciar sesión se abre el dashboard **"Tarea K8s - Grafana con Deployment y ReplicaSet"**. En *Connections → Data sources* aparece la fuente **TestData**.

### 6. Reproducir las pruebas del informe

Todos los comandos van en una segunda terminal, dentro de la carpeta del repositorio.

**a) Relación Deployment → ReplicaSet → Pod**

```bash
kubectl -n grafana-lab get rs -o custom-columns=REPLICASET:.metadata.name,DUENO:.metadata.ownerReferences[0].kind,NOMBRE-DUENO:.metadata.ownerReferences[0].name
```

```bash
kubectl -n grafana-lab get pods -o custom-columns=POD:.metadata.name,DUENO:.metadata.ownerReferences[0].kind,NOMBRE-DUENO:.metadata.ownerReferences[0].name
```

**b) Autorreparación.** Se borra un Pod (el nombre sale de `get pods`) y el ReplicaSet crea otro:

```bash
kubectl -n grafana-lab get pods
```

```bash
kubectl -n grafana-lab delete pod <nombre-de-un-pod>
```

```bash
kubectl -n grafana-lab get pods
```

**c) Escalado**

```bash
kubectl -n grafana-lab scale deploy grafana --replicas=5
```

```bash
kubectl -n grafana-lab get deploy,rs,pods
```

```bash
kubectl -n grafana-lab scale deploy grafana --replicas=3
```

**d) Actualización continua (RollingUpdate)**

```bash
kubectl -n grafana-lab set image deploy/grafana grafana=grafana/grafana:12.0.0
```

```bash
kubectl -n grafana-lab rollout status deploy/grafana
```

```bash
kubectl -n grafana-lab get rs -o wide
```

```bash
kubectl -n grafana-lab rollout history deploy/grafana
```

**e) Rollback.** El *Warning* sobre `kubectl apply` que aparece es informativo:

```bash
kubectl -n grafana-lab rollout undo deploy/grafana
```

```bash
kubectl -n grafana-lab get rs -o wide
```

**f) ReplicaSet sin Deployment.** La plantilla cambia a 12.0.0, pero los Pods siguen en 11.6.0:

```bash
kubectl apply -f grafana/05-replicaset-demo.yml
```

```bash
kubectl -n grafana-lab set image rs/grafana-rs-demo grafana=grafana/grafana:12.0.0
```

```bash
kubectl -n grafana-lab get pods -l app=grafana-rs-demo -o custom-columns=POD:.metadata.name,IMAGEN:.spec.containers[0].image
```

```bash
kubectl delete -f grafana/05-replicaset-demo.yml
```

### Solución de problemas

| Problema | Causa / solución |
|---|---|
| `Unable to connect to the server ... localhost:8080` | El clúster está apagado: ejecutar `minikube start -p taller`. |
| `failed to connect to the docker API` | Docker Desktop no está encendido. |
| Pods en `ContainerCreating` o `0/1` por varios minutos | Se está descargando la imagen o Grafana aún arranca; esperar con `kubectl -n grafana-lab rollout status deploy/grafana`. |
| `http://192.168.49.2:30300` no abre en Windows/macOS | Con el driver Docker el nodo no es accesible directamente; usar `minikube service ...` y dejar la terminal abierta. |
| Grafana pide iniciar sesión otra vez después de un rollout | Es esperado: cada réplica guarda su sesión localmente y los Pods nuevos no la conocen. |

## Pruebas realizadas

Las salidas completas están en [`evidencias/`](evidencias/).

| # | Prueba | Comandos clave | Resultado | Evidencia |
|---|---|---|---|---|
| 1 | Recursos creados | `kubectl -n grafana-lab get deploy,rs,pods,svc -o wide` | 1 Deployment, 1 ReplicaSet, 3 Pods `Running`, Service NodePort `80:30300` | [01](evidencias/01-recursos.txt) |
| 2 | Relación Deployment → ReplicaSet → Pod | `get rs/pods -o custom-columns=...ownerReferences...` | El ReplicaSet es propiedad del Deployment y los Pods del ReplicaSet | [02](evidencias/02-relacion-deploy-rs-pod.txt) |
| 3 | Autorreparación | `kubectl -n grafana-lab delete pod <pod>` | En 2 s el ReplicaSet creó un Pod nuevo y volvió a 3/3 | [03](evidencias/03-autorreparacion.txt) |
| 4 | Escalado | `kubectl -n grafana-lab scale deploy grafana --replicas=5` (luego 2 y 3) | El mismo ReplicaSet subió a 5, bajó a 2 y volvió a 3 | [04](evidencias/04-escalado.txt) |
| 5 | Rolling update | `kubectl -n grafana-lab set image deploy/grafana grafana=grafana/grafana:12.0.0` | Se creó un ReplicaSet nuevo (12.0.0) con 3 Pods; el viejo quedó en 0 sin caída | [05](evidencias/05-rolling-update-rollback.txt) |
| 6 | Rollback | `kubectl -n grafana-lab rollout undo deploy/grafana` | Se reactivó el ReplicaSet original (11.6.0); historial con revisiones | [05](evidencias/05-rolling-update-rollback.txt) |
| 7 | ReplicaSet sin Deployment | `kubectl apply -f grafana/05-replicaset-demo.yml` y `set image rs/...` | Se autorrepara, pero al cambiar la imagen los Pods siguen en 11.6.0 y no hay historial | [06](evidencias/06-replicaset-solo.txt) |
| 8 | Exposición | `kubectl -n grafana-lab get svc,endpointslices` | El Service apunta a las 3 IPs de los Pods; Grafana responde por el túnel | [07](evidencias/07-exposicion.txt) |

## Usos de Grafana desplegado con Deployment

- **Monitoreo de infraestructura y de Kubernetes:** junto con Prometheus, dashboards de CPU, memoria, Pods y nodos del propio clúster.
- **Observabilidad de aplicaciones:** métricas (Prometheus), logs (Loki) y trazas (Tempo) en un solo lugar.
- **Alertas:** reglas que avisan por correo, Slack o Teams cuando una métrica cruza un umbral.
- **Dashboards de negocio:** conexión directa a MySQL, PostgreSQL o SQL Server para ventas, inventario o KPIs.
- **IoT y series de tiempo:** sensores guardados en InfluxDB o TimescaleDB.
- **Centro de operaciones (NOC):** pantallas compartidas con el estado de los servicios.

**Qué aporta el Deployment en estos usos:**
- **Alta disponibilidad:** si una réplica cae, el ReplicaSet la reemplaza y el Service sigue respondiendo con las demás.
- **Actualizaciones sin caída:** se cambia la versión de Grafana con RollingUpdate y se puede volver atrás si algo falla.
- **Escalado:** más réplicas cuando muchos usuarios consultan dashboards al mismo tiempo.
- **Configuración como código:** Secret y ConfigMap hacen que todas las réplicas arranquen iguales y que el despliegue se pueda repetir en otro clúster.

**Limitación y cómo se resuelve en producción:** en este laboratorio cada réplica guarda sus datos en un `emptyDir` (SQLite local), por eso los cambios hechos desde la interfaz no se comparten entre réplicas ni sobreviven a un reinicio. En producción se conecta Grafana a una base de datos externa (PostgreSQL o MySQL) con `GF_DATABASE_*`, y los dashboards se mantienen como código en ConfigMaps, como se hizo aquí.

## Limpieza

```bash
kubectl delete namespace grafana-lab
```

```bash
minikube stop -p taller
```

Para borrar por completo el clúster de prueba:

```bash
minikube delete -p taller
```
