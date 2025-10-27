# Práctica final - Contenedores: más que VMs - Kubernetes
**XII Edición Bootcamp DevOps & Cloud Computing Full Stack**

**Evaristo García Zambrana | 21 de septiembre de 2025**

---

[🔽 Ir directamente a Cómo desplegar kc-wordpress-k8s 🔽](#-c%C3%B3mo-desplegar-kc-wordpress-k8s)

## Descripción
*kc-wordpress-k8s* es un pequeño laboratorio de Kubernetes gestionado a través de Helm Chart en un entorno local con minikube. Éste despliega, al menos, dos réplicas de WordPress junto a una base de datos MariaDB.

Tiene como finalidad entender, comprender y poner en práctica cómo funcionan y cómo se configuran los objetos de Kubernetes, así como sus posibilidades para implementar una infraestructura de microservicios.

## Diagrama de arquitectura
Diagrama que contempla el resultado final de la arquitectura de microservicios desplegada por el Helm Chart de este repositorio.

> [!NOTE]
> Esta arquitectura no es la más idónea. Al usar minikube hay una limitación a nivel de almacenamiento, donde los StorageClass no dan RWX ni compartición entre nodos. Esto obliga a que todos los pods de WordPress se desplieguen en el mismo nodo y así se ha hecho mediante una regla de afinidad, para que pueda consultarse los datos de /wp-content/uploads.
> A nivel de base de datos tampoco lo sería, puesto que habría que crear un clúster de MariaDB para que los datos estuviesen sincronizados a nivel de aplicación.

![Diagrama de arquitectura de la aplicación desplegada en Kubernetes](https://github.com/EvaristoGZ/Kubernetes-Helm-Chart/blob/main/01%20-%20Diagrama%20Kubernetes%20-%20Evaristo%20GZ.drawio.jpg)

## Requisitos
- Docker Engine (o Docker Desktop)
- Git
- Minikube
- kubectl
- Helm

Ejecutado en Windows 11 con Docker Desktop 4.46.0, Docker Engine 28.4.0, minikube v1.36.0, kubectl v1.34.0 y Helm v3.18.6 mediante WSL 2.

> [!NOTE]
> Se omite las instrucciones de instalación de los componentes anteriores, recomendándose seguir la documentación oficial de cada uno de ellos.

## Estructura del repositorio
```
kc-wordpress-k8s/                   <- Directorio principal que contiene el chart Helm
├── Chart.yaml                      <- Información sobre el chart
├── templates                       <- Directorio con las plantillas de Helm
│   ├── _helpers.tpl                <- Funciones reutilizables en el resto de plantillas
│   ├── database-secret.yaml        <- Guarda las cadenas de conexión de BBDD en Secrets
│   ├── mariadb-statefulset.yaml    <- Define y despliega pod de MariaDB
│   ├── mariadb-service.yaml        <- Crea servicio para MariaDB (puerto 3306)
│   ├── wordpress-deployment.yaml   <- Define y despliega pods (2min 5máx) de WordPress
│   ├── wordpress-hpa.yaml          <- Define reglas de autoescaler por uso de CPU
│   ├── wordpress-ingress.yaml      <- Despliega ingress con valores de values.yaml
│   ├── wordpress-pdb.yaml          <- Define el PodDisruptionBudget para WordPress
│   ├── wordpress-pvc.yaml          <- Crea volumen persistente para MariaDB
│   └── wordpress-service.yaml      <- Crea servicio para WordPress (puerto 80)
└── values.yaml                     <- Define valores usados para los templates
```

## 🚀 Cómo desplegar kc-wordpress-k8s

### 1. Clona el repositorio de GitHub
```
git clone https://github.com/KeepCodingCloudDevops12/EvaristoGZ_03-Kubernetes
```

### 2. Ubícate en el directorio Git
```
cd EvaristoGZ_03-Kubernetes
```

### 3. Arranca Docker Desktop, minikube y habilita sus addons
Primeramente, arranca Docker Desktop arranca el clúster de minikube con, al menos, dos nodos.
```
minikube start --nodes 2 ; minikube addons enable metrics-server ; minikube addons enable ingress
```

### 4. Habilita el acceso externo
En una nueva terminal de WSL, habilita el túnel. Requerirá privilegios de sudo.
```
minikube tunnel
```

### 5. Edita tu fichero hosts
Edita el fichero host de tu sistema, en este caso *C:\Windows\System32\drivers\etc\hosts* en Windows y añade
```
127.0.0.1 evaristogz.local.k8s
```

### 6. Instala el Helm Chart
Consulta los valores personalizables [aquí](#-valores-personalizables).
```
helm upgrade --install kc-wordpress-k8s ./ -n keepcoding --create-namespace \
-f values.yaml \
--set mariadb.env.rootPassword=ContrasenyaROOT \
--set mariadb.env.password=ContrasenyaWP \
--set mariadb.env.user=UsuarioWP
```

### 7. Accede a través del navegador web
Accede a http://evaristogz.local.k8s a través de un navegador web.

El resultado esperado es el asistente de instalación de WordPress.

## Valores personalizables
Puedes personalizar los valores de los parámetros editando el *values.yaml* o utilizando `--set nivel1.nivel2.nivel3=valor`.

### MariaDB
| Parámetro          | Descripción                                             | Valor por defecto        |
|--------------------|---------------------------------------------------------|--------------------------|
| `image`            | Imagen de contenedor utilizada para la base de datos    | `mariadb:12.0.2`         |
| `storageClassName` | StorageClass a usar para el PVC                         | `standard`               |
| `pvcSize`          | Tamaño del volumen persistente                          | `1Gi`                    |
| `env.secretName`   | Nombre del Secret de Kubernetes con credenciales        | `database-connection`    |
| `env.rootPassword` | Contraseña de root de la base de datos                  | `rootpassworddefault`    |
| `env.database`     | Nombre de la base de datos para WordPress               | `wordpress`              |
| `env.user`         | Usuario de la base de datos para WordPress              | `wpusuariodefault`       |
| `env.password`     | Contraseña del usuario de la base de datos de WordPress | `wpcontrasenadefault`    |



### WordPress
| Parámetro                                              | Descripción                                                  | Valor por defecto           |
|--------------------------------------------------------|--------------------------------------------------------------|-----------------------------|
| `wordpress.replicas`                                   | Número de réplicas iniciales del pod de WordPress            | `2`                         |
| `wordpress.image`                                      | Imagen de contenedor utilizada para WordPress                | `wordpress:php8.4-apache`   |
| `wordpress.storageClassName`                           | StorageClass a usar para el PVC                              | `standard`                  |
| `wordpress.pvcSize`                                    | Tamaño del volumen persistente para `wp-content`             | `1Gi`                       |
| `wordpress.service.type`                               | Tipo de servicio de Kubernetes                               | `ClusterIP`                 |
| `wordpress.service.port`                               | Puerto expuesto por el servicio                              | `80`                        |
| `wordpress.resources.requests.cpu`                     | CPU mínima solicitada por el contenedor                      | `100m`                      |
| `wordpress.resources.requests.memory`                  | Memoria mínima solicitada por el contenedor                  | `128Mi`                     |
| `wordpress.resources.limits.cpu`                       | Límite máximo de CPU permitido                               | `250m`                      |
| `wordpress.resources.limits.memory`                    | Límite máximo de memoria permitido                           | `256Mi`                     |
| `wordpress.autoscaling.enabled`                        | Habilita el Horizontal Pod Autoscaler (HPA)                  | `true`                      |
| `wordpress.autoscaling.minReplicas`                    | Número mínimo de réplicas                                    | `2`                         |
| `wordpress.autoscaling.maxReplicas`                    | Número máximo de réplicas                                    | `5`                         |
| `wordpress.autoscaling.targetCPUUtilizationPercentage` | Umbral de utilización de CPU para escalar                    | `70`                        |
| `wordpress.ingress.enabled`                            | Habilita el recurso Ingress                                  | `true`                      |
| `wordpress.ingress.host`                               | Hostname que expone WordPress mediante Ingress               | `evaristogz.local.k8s`      |
| `wordpress.ingress.className`                          | Controlador de Ingress que manejará el tráfico               | `nginx`                     |


# Desplegar múltiples veces la aplicación
Si, por cualquier motivo, deseas reutilizar el Helm Chart para desplegar más de una vez la aplicación en el mismo namespace, bastará con establecer valores distintos en *mariadb.env.secretName* (*database-connection* en el despliegue de este README) y en *wordpress.ingress.host* (*evaristogz.local.k8s* en el despliegue de este README).

El valor que indiques en *wordpress.ingress.host* también deberá ser incluido el fichero */etc/hosts*

Por ejemplo:
```
helm upgrade --install kc-wordpress-k8s ./ -n keepcoding --create-namespace \
-f values.yaml \
mariadb.env.secretName=conexion-bbdd \
--set wordpress.ingress.host=wp.entorno.local
```

## Comandos y depuración
### Desinstalar el chart de Helm
`helm uninstall -n keepcoding kc-wordpress-k8s`

### Eliminar clúster minikube
`minikube delete`

### Ver todos los pods del namespace
`kubectl -n keepcoding get pods`

### Ver todos los pods del namespace y su nodo
`kubectl -n keepcoding get pods -o wide`

### Ver información extensa de un pod
`kubectl -n keepcoding describe pod NOMBREPOD`

### Ver los logs de un pod
`kubectl -n keepcoding logs NOMBREPOD`

### Ver todos los servicios del namespace
`kubectl -n keepcoding get svc`

### Ver todos los PVC del namespace
`kubectl -n keepcoding get pvc`

### Ver todos los Secrets del namespace
`kubectl -n keepcoding get secrets`

### Ver valor de un secreto
`kubectl -n keepcoding get secret NOMBRESECRETO -o jsonpath='{.nivel1.nivel2}' | base64 -d`

Donde *nivel1.nivel2* puede ser *.data.MARIADB_PASSWORD*, por ejemplo.

## Errores comunes
### Error de conexión de base de datos
```
# Verificar que MariaDB esté funcionando
kubectl -n keepcoding get pods -l app.kubernetes.io/component=mariadb

# Verificar logs de MariaDB
kubectl -n keepcoding logs deployment/[RELEASE_NAME]-mariadb

# Verificar el secreto de la base de datos
kubectl -n keepcoding get secret database-connection -o yaml

# Común: MariaDB no ha terminado de inicializarse
# Solución: Esperar a que MariaDB esté completamente listo
kubectl -n keepcoding wait --for=condition=ready pod -l app.kubernetes.io/component=mariadb --timeout=300s
```

### Problema con el almacenamiento persistente
```
# Verificar PVCs
kubectl -n keepcoding get pvc

# Verificar que la StorageClass exista
kubectl -n keepcoding get storageclass
```

### No puedo acceder a la página web / 503 Service Temporarily Unavailable
```
# Verificar que los pods de WordPress estén "Ready"
kubectl -n keepcoding get pods

# Verificar que el Ingress Controller esté instalado
kubectl -n keepcoding get pods -n ingress-nginx

# Verificar el Ingress
kubectl -n keepcoding get ingress

# Verificar que el host esté configurado en /etc/hosts la línea:
127.0.0.1 evaristogz.local.k8s
```

### Error ERR_CONNECTION_REFUSED
```
# Verificar que el Minikube Tunnel está corriendo
En ocasiones, es posible que se requiera ingresar la contraseña de sudo nuevamente para seguir corriendo el túnel.

# Verificar que Minikube Tunnel reconoce el servicio
Cuando instalas el Helm Chart, aparecerá un mensaje del tipo: `🏃  Starting tunnel for service` en la terminal que levantamos el túnel.

```

### No puedo loguearme o me pide inicio de sesión constantemente
Si no puedes loguearte tras instalar WordPress a través de http://evaristogz.local.k8s, prueba habilitar un service con
`minikube service -n keepcoding kc-wordpress-k8s-wordpress` esto te habilitará una dirección URL del tipo http://127.0.0.1:XXXXX desde la que podrás acceder.

Es posible que, de esta forma, se te pida hacer login en el backoffice con cierta frecuencia.

Esto es debido al uso de cookies y siteurl que requieren de una configuración más extensa para solucionarlo.
