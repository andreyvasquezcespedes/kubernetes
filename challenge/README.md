[# Kubernetes Challenge – Teemii

## 1. Descripción general

Este proyecto tiene como objetivo migrar una aplicación originalmente desplegada mediante Docker Compose hacia un entorno orquestado con Kubernetes, utilizando Minikube como plataforma local de ejecución.

Como parte del challenge se implementan los componentes necesarios para desplegar la aplicación, administrar sus recursos, exponer sus servicios y establecer una plataforma de monitoreo y observabilidad.

El proyecto también incorpora herramientas del ecosistema Grafana para recopilar, almacenar, consultar y visualizar los registros generados por los contenedores.

### Objetivos

- Preparar un entorno local de Kubernetes.
- Migrar la aplicación Teemii desde Docker Compose hacia Kubernetes.
- Implementar Deployments y Services.
- Configurar almacenamiento persistente.
- Exponer la aplicación mediante servicios de Kubernetes.
- Habilitar métricas del clúster.
- Instalar Prometheus y Grafana.
- Implementar Loki para almacenamiento y consulta de logs.
- Configurar Grafana Alloy para recopilar los registros de los pods.
- Construir un dashboard para visualizar los logs.

---

## 2. Arquitectura del proyecto

La solución utiliza una máquina virtual Ubuntu ejecutada mediante Vagrant y VirtualBox. Dentro de ella se ejecuta Minikube, utilizando Docker como entorno de ejecución del clúster.

La aplicación Teemii está compuesta por dos servicios principales:

- **Frontend:** interfaz web de la aplicación.
- **Backend:** servicio encargado de la lógica y procesamiento de la aplicación.

La plataforma de observabilidad utiliza Prometheus, Grafana, Loki y Alloy.

### Tecnologías utilizadas

| Categoría | Tecnología |
|---|---|
| Sistema operativo | Ubuntu 22.04 |
| Virtualización | VirtualBox |
| Administración de VM | Vagrant |
| Contenedores | Docker |
| Orquestación | Kubernetes |
| Clúster local | Minikube |
| Administración de paquetes | Helm |
| Métricas | Prometheus |
| Visualización | Grafana |
| Logs | Loki |
| Recolección de logs | Grafana Alloy |
| Métricas del clúster | Metrics Server |

---

# 3. Desarrollo del challenge

## Parte 1. Preparación del entorno

Se preparó el entorno de trabajo para ejecutar Kubernetes localmente.

### Componentes instalados

- VirtualBox.
- Vagrant.
- Ubuntu 22.04 (Jammy).
- Docker Engine.
- Minikube.
- kubectl.
- Helm.

El clúster se ejecuta dentro de la máquina virtual Ubuntu, evitando depender de una instalación directa de Kubernetes sobre el sistema operativo anfitrión.

### Verificación del entorno

```bash
docker --version
```

```bash
kubectl version --client
```

```bash
minikube version
```

```bash
helm version
```

### Iniciar Minikube

```bash
minikube start --driver=docker
```

Verificar el estado:

```bash
minikube status
```

```bash
kubectl get nodes -o wide
```

El entorno utilizado durante las pruebas quedó con:

- Minikube 1.37.0.
- Kubernetes 1.37.0.
- Un nodo de control `minikube`.
- Container runtime: containerd.

### Habilitar Ingress

Se trabajó con el complemento Ingress de Minikube para habilitar el enrutamiento HTTP dentro del clúster.

```bash
minikube addons enable ingress
```

Verificar los recursos:

```bash
kubectl get pods -n ingress-nginx
```

```bash
kubectl get services -n ingress-nginx
```

---

## Parte 2. Migración de Teemii a Kubernetes

La aplicación Teemii originalmente utilizaba Docker Compose para ejecutar sus componentes.

El objetivo de esta etapa fue trasladar esa arquitectura a recursos nativos de Kubernetes.

### Componentes de la aplicación

| Componente | Imagen | Puerto |
|---|---|---|
| Frontend | `dokkaner/teemii-frontend:develop` | 80 |
| Backend | `dokkaner/teemii-backend:develop` | 3000 y 1555 |

Se utilizaron las imágenes publicadas existentes, por lo que inicialmente no fue necesario construir nuevas imágenes ni publicarlas en un registro.

### Namespace de la aplicación

Se creó un namespace independiente para mantener organizados los recursos de Teemii.

```bash
kubectl create namespace teemii
```

Verificar:

```bash
kubectl get namespaces
```

### Backend

Se desplegó el backend mediante un Deployment con una réplica.

El servicio se expuso internamente mediante un Service de tipo ClusterIP.

Características:

- Deployment: backend.
- Réplicas: 1.
- Service: ClusterIP.
- Puertos: 3000 y 1555.

El Service ClusterIP permite que otros componentes del clúster accedan al backend sin exponerlo directamente al exterior.

### Frontend

El frontend se desplegó mediante un Deployment con dos réplicas.

Características:

- Deployment: frontend.
- Réplicas: 2.
- Service: NodePort.
- Puerto del servicio: 8080.
- Puerto del nodo: 30080.
- Puerto del contenedor: 80.

La configuración permite acceder al frontend mediante el puerto NodePort del nodo Minikube.

### Variables de entorno

Se conservaron las variables utilizadas por la aplicación:

```yaml
VITE_APP_TITLE: Teemii
VITE_APP_PORT: "80"
```

### Almacenamiento persistente

Se incorporó un PersistentVolumeClaim denominado:

```text
teemii-data
```

El almacenamiento se utiliza para conservar los datos de la aplicación mediante un volumen montado en:

```text
/app/data
```

El tamaño solicitado en la configuración inicial fue de 5Gi.

### Aplicar los manifiestos

Los manifiestos de Kubernetes se almacenan en el directorio de configuración del proyecto.

Ejemplo de aplicación:

```bash
kubectl apply -f k8s/
```

Verificar los recursos:

```bash
kubectl get all -n teemii
```

```bash
kubectl get pvc -n teemii
```

```bash
kubectl get pods -n teemii -o wide
```

### Acceso a Teemii

El frontend se configuró mediante NodePort.

Consultar la IP de Minikube:

```bash
minikube ip
```

El acceso se realiza utilizando:

```text
http://IP_DE_MINIKUBE:30080
```

También se puede utilizar port-forward:

```bash
kubectl port-forward -n teemii \
  svc/teemii-frontend 8080:8080
```

Y acceder desde el navegador:

```text
http://127.0.0.1:8080
```

---

## Parte 3. Métricas y administración del clúster

En esta etapa se habilitó Metrics Server para obtener métricas de utilización de CPU y memoria de los nodos y pods.

### Habilitar Metrics Server

```bash
minikube addons enable metrics-server
```

Verificar su estado:

```bash
kubectl get pods -n kube-system
```

Comprobar las métricas de los nodos:

```bash
kubectl top nodes
```

Comprobar las métricas de los pods de Teemii:

```bash
kubectl top pods -n teemii
```

Metrics Server permite consultar el consumo actual de recursos y proporciona la API de métricas utilizada por Kubernetes.

### Verificar la API de métricas

```bash
kubectl get apiservices | grep metrics
```

La API esperada es:

```text
v1beta1.metrics.k8s.io
```

---

## Parte 4. Monitoreo y observabilidad

Esta etapa incorpora una plataforma para supervisar el clúster y consultar los registros de las aplicaciones.

Se utilizaron dos herramientas principales:

- Prometheus y Grafana para métricas y visualización.
- Loki y Grafana Alloy para recopilación y consulta de logs.

### 4.1. Crear el namespace de monitoreo

Se creó un namespace independiente denominado `monitoring`.

```bash
kubectl create namespace monitoring
```

Verificar:

```bash
kubectl get namespaces
```

> El namespace utilizado en la implementación final es `monitoring`.

### 4.2. Instalar Helm

Se utilizó Helm para administrar las instalaciones de los componentes de observabilidad.

Consultar la versión:

```bash
helm version
```

Agregar el repositorio de Prometheus:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
```

Actualizar repositorios:

```bash
helm repo update
```

### 4.3. Instalar Prometheus y Grafana

Se utilizó el chart:

```text
prometheus-community/kube-prometheus-stack
```

Versión del chart:

```text
93.11.0
```

Release:

```text
prometheus
```

Namespace:

```text
monitoring
```

Instalación:

```bash
helm upgrade --install prometheus \
  prometheus-community/kube-prometheus-stack \
  --version 93.11.0 \
  --namespace monitoring \
  --values prometheus/values.yaml \
  --wait
```

El stack incorpora los siguientes componentes:

| Componente | Función |
|---|---|
| Prometheus | Recopilación y consulta de métricas |
| Grafana | Visualización de métricas |
| Prometheus Operator | Administración de recursos de monitoreo |
| kube-state-metrics | Métricas del estado de Kubernetes |
| Node Exporter | Métricas del sistema operativo y nodos |

Se configuró almacenamiento persistente para Grafana y Prometheus:

- Grafana: 1Gi.
- Prometheus: 2Gi.
- Retención de Prometheus: 3 días.

Alertmanager se dejó deshabilitado en la configuración utilizada.

### Verificar la instalación

```bash
helm list -n monitoring
```

```bash
kubectl get pods -n monitoring
```

```bash
kubectl get pvc -n monitoring
```

### Acceso a Grafana

Se utilizó el servicio de Grafana desplegado por Helm.

Para consultar el nombre del servicio:

```bash
kubectl get svc -n monitoring
```

Se puede acceder mediante port-forward:

```bash
kubectl port-forward -n monitoring \
  svc/prometheus-grafana 3000:80
```

Abrir:

```text
http://127.0.0.1:3000
```

Las credenciales iniciales se consultan desde el Secret generado por el chart, de acuerdo con la configuración instalada.

### Acceso a Prometheus

Para consultar Prometheus:

```bash
kubectl port-forward -n monitoring \
  svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Abrir:

```text
http://127.0.0.1:9090
```

Si se accede desde el equipo anfitrión y el port-forward se ejecuta dentro de Vagrant, se puede utilizar un túnel SSH:

```bash
vagrant ssh -- -L 9090:127.0.0.1:9090
```

### 4.4. Instalar Loki

Loki se incorporó como sistema de almacenamiento y consulta de registros.

Se agregó el repositorio utilizado:

```bash
helm repo add grafana-community \
  https://grafana-community.github.io/helm-charts
```

Actualizar repositorios:

```bash
helm repo update
```

Se utilizó el chart Loki versión `21.6.0`, con configuración adaptada al entorno local de Minikube.

La instalación se realizó en el namespace:

```text
monitoring
```

El servicio de Loki quedó disponible internamente mediante:

```text
http://loki.monitoring.svc.cluster.local:3100
```

Verificar:

```bash
kubectl get pods -n monitoring
```

```bash
kubectl get svc -n monitoring
```

### Verificar que Loki está listo

```bash
kubectl port-forward -n monitoring svc/loki 3100:3100
```

En otra terminal:

```bash
curl http://127.0.0.1:3100/ready
```

### 4.5. Instalar Grafana Alloy

Grafana Alloy se utilizó como agente para descubrir los pods de Kubernetes, leer sus archivos de logs y enviarlos a Loki.

Se utilizó el chart:

```text
grafana/alloy
```

Versión:

```text
1.16.0
```

Se configuró como DaemonSet para recopilar registros desde los nodos.

La configuración contempla:

- Descubrimiento de pods mediante Kubernetes.
- Identificación del namespace, pod y contenedor.
- Lectura de archivos de logs desde `/var/log/pods`.
- Procesamiento del formato CRI.
- Envío de registros hacia Loki.

### Configuración de Alloy

El archivo principal de configuración se denomina:

```text
config.alloy
```

Se utiliza un ConfigMap denominado:

```text
alloy-config
```

El endpoint de Loki configurado en Alloy es:

```text
http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push
```

La configuración de lectura de archivos incluye el bloque `file_match`:

```hcl
loki.source.file "pod_logs" {
  targets    = discovery.relabel.pod_logs.output
  forward_to = [loki.process.pod_logs.receiver]

  file_match {
    enabled     = true
    sync_period = "10s"
  }
}
```

Este ajuste permitió resolver los errores de lectura asociados a los patrones de archivos de logs de los contenedores.

### Aplicar el ConfigMap

```bash
kubectl create configmap alloy-config \
  --namespace monitoring \
  --from-file=config.alloy \
  --dry-run=client -o yaml > configmap.yaml
```

Aplicar la configuración:

```bash
kubectl apply -f configmap.yaml
```

Reiniciar Alloy:

```bash
kubectl rollout restart daemonset/alloy \
  -n monitoring
```

Verificar el despliegue:

```bash
kubectl rollout status daemonset/alloy \
  -n monitoring
```

### Verificar los logs de Alloy

```bash
kubectl get pods -n monitoring
```

Consultar los registros del agente:

```bash
kubectl logs -n monitoring \
  -l app.kubernetes.io/name=alloy \
  -c alloy \
  --since=5m
```

Para identificar errores:

```bash
kubectl logs -n monitoring \
  -l app.kubernetes.io/name=alloy \
  -c alloy \
  --since=5m | grep -E 'level=error|failed'
```

---

## 5. Integración de Loki con Grafana

Una vez instalado Loki, se configuró como fuente de datos de Grafana.

### Configuración del Data Source

En Grafana:

1. Abrir Connections.
2. Seleccionar Data sources.
3. Agregar un nuevo Data Source.
4. Seleccionar Loki.
5. Configurar la URL.

```text
http://loki.monitoring.svc.cluster.local:3100
```

6. Guardar mediante Save & test.

La configuración también puede declararse mediante Helm utilizando `grafana.additionalDataSources` en los valores de `kube-prometheus-stack`.

---

## 6. Validación de la recopilación de logs

Se realizaron consultas directamente contra la API de Loki para confirmar que los registros estaban llegando correctamente.

### Consulta de logs

Para consultar registros se utilizó el endpoint:

```text
/loki/api/v1/query_range
```

Ejemplo:

```bash
curl -G -s \
  'http://127.0.0.1:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={namespace="monitoring", container="grafana"}' \
  --data-urlencode 'limit=10' \
  --data-urlencode 'since=15m'
```

La consulta devolvió registros reales del contenedor Grafana.

Se verificaron etiquetas como:

- namespace.
- pod.
- container.
- filename.
- stream.
- detected_level.

Esto permitió confirmar el flujo de recopilación:

```text
Pods Kubernetes
       |
       v
Grafana Alloy
       |
       v
Loki
       |
       v
Grafana
```

La consulta del namespace de prueba `todo-app` devolvió resultados vacíos porque era un entorno temporal y no formaba parte de la aplicación final.

---

## 7. Dashboard de logs en Grafana

Se creó un dashboard para visualizar los registros de los contenedores.

Nombre sugerido:

```text
Kubernetes Logs Dashboard
```

El dashboard utiliza Loki como fuente de datos.

### Consulta general

```logql
{namespace="monitoring"}
```

### Logs de Grafana

```logql
{namespace="monitoring", container="grafana"}
```

### Logs de Alloy

```logql
{namespace="monitoring", container="alloy"}
```

### Filtrar errores

```logql
{namespace="monitoring"} |~ "(?i)error|failed|fatal"
```

### Filtrar advertencias

```logql
{namespace="monitoring"} |~ "(?i)warn|warning"
```

Se recomienda exportar el dashboard en formato JSON para conservarlo como parte del repositorio y permitir su posterior importación.

---

## 8. Comandos de administración y diagnóstico

### Estado general del clúster

```bash
kubectl get nodes -o wide
```

```bash
kubectl get namespaces
```

### Aplicación Teemii

```bash
kubectl get all -n teemii
```

```bash
kubectl get pvc -n teemii
```

```bash
kubectl logs -n teemii deployment/teemii-backend
```

### Observabilidad

```bash
kubectl get all -n monitoring
```

```bash
kubectl get pvc -n monitoring
```

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

### Helm

```bash
helm list -n monitoring
```

```bash
helm status prometheus -n monitoring
```

### Reiniciar Alloy

```bash
kubectl rollout restart daemonset/alloy \
  -n monitoring
```

### Consultar eventos

```bash
kubectl get events -n monitoring \
  --sort-by='.lastTimestamp'
```

---

## 9. Estructura de archivos recomendada

La documentación y los manifiestos se mantienen dentro del directorio `challenge`.

Una organización recomendada para el repositorio es:

```text
challenge/
│
├── README.md
│
├── k8s/
│   ├── namespace.yaml
│   ├── backend/
│   ├── frontend/
│   ├── storage/
│   └── monitoring/
│       ├── prometheus/
│       │   └── values.yaml
│       │
│       ├── loki/
│       │   └── values.yaml
│       │
│       └── alloy/
│           ├── values.yaml
│           ├── config.alloy
│           └── configmap.yaml
│
└── docs/
    ├── arquitectura.md
    └── evidencias/
```

> Esta estructura representa una organización recomendada para documentar los recursos. Los nombres y ubicaciones definitivos deben coincidir con los archivos existentes en el repositorio.

Se recomienda conservar también el JSON exportado del dashboard de Grafana.

No se deben versionar contraseñas, tokens, certificados privados ni archivos con credenciales.

---

## 10. Limpieza del entorno

Para eliminar los recursos de la aplicación:

```bash
kubectl delete namespace teemii
```

Para eliminar los componentes de observabilidad:

```bash
helm uninstall prometheus -n monitoring
```

Loki y Alloy deben eliminarse mediante los releases Helm correspondientes a los nombres utilizados durante su instalación.

Si se desea eliminar todo el namespace de monitoreo:

```bash
kubectl delete namespace monitoring
```

Para detener Minikube:

```bash
minikube stop
```

Para eliminar el clúster local:

```bash
minikube delete
```

> La eliminación de namespaces y del clúster puede borrar los recursos y datos asociados. Ejecutar estos comandos únicamente cuando se desee destruir el entorno.

---

## 11. Resultado final

Al finalizar las actividades documentadas se dispone de:

- Un entorno local de Kubernetes basado en Minikube.
- Una aplicación Teemii migrada desde Docker Compose.
- Frontend y backend desplegados como recursos de Kubernetes.
- Servicios internos y exposición del frontend mediante NodePort.
- Almacenamiento persistente para la aplicación.
- Metrics Server para consultar el consumo de recursos.
- Prometheus para recopilar métricas.
- Grafana para visualizar métricas y logs.
- Loki para almacenar y consultar registros.
- Grafana Alloy para recopilar logs de los contenedores.
- Un dashboard de Grafana para consultar los registros del clúster.

### Flujo final de observabilidad

```text
                    KUBERNETES
                        |
             +----------+----------+
             |                     |
          MÉTRICAS                LOGS
             |                     |
         Prometheus            Alloy
             |                     |
             |                   Loki
             |                     |
             +----------+----------+
                        |
                     Grafana
                        |
                  Dashboards
```

---

## 12. Conclusión

Este challenge permitió desarrollar una implementación práctica de Kubernetes, comenzando por la preparación de un entorno local y la migración de una aplicación basada en Docker Compose.

Posteriormente se incorporaron herramientas de monitoreo y observabilidad para obtener información sobre el estado del clúster y los registros generados por los contenedores.

La solución demuestra cómo integrar Kubernetes, Helm, Prometheus, Grafana, Loki y Alloy para construir una plataforma local de despliegue y observabilidad.

El proyecto queda documentado como una base reproducible para continuar experimentando con administración de clústeres, monitoreo, almacenamiento, automatización y prácticas DevOps.
