# kubernetes-what

Parte práctica de la presentación Kubernetes What? La idea es que los asistentes puedan experimentar con un cluster de Kubernetes y prueben algunos de los conceptos que se ha visto durante la charla.

## Prerrequisitos

Para poder realizar los diferentes ejercicios es necesario tener acceso a un cluster de Kubernetes. Lo más conveniente es [instalar Minikube](https://minikube.sigs.k8s.io/docs/start/), solo es necesario tener Docker y podrás tener un cluster local de Kubernetes.

Una vez se tiene Minikube funcionando, vamos a instalar un addon y a crear un Namespace para los ejercicios:
```bash
minikube addons enable ingress
kubectl create namespace kubernetes-what
```

Para el último ejercicio necesitaremos también tener instalado [Helm](https://helm.sh/docs/intro/install/).

## Desplegando y exponiendo un Pod (Con comandos)

Empecemos desplegando un Pod de Nginx en el cluster, pese a que no es habitual desplegar un Pod por si solo, para realizar pruebas es útil.
```bash
pi@raspberrypi:~ $ kubectl run nginx --image=nginx -n kubernetes-what
pod/nginx created
```

Al revisar el estado del Pod, deberíamos ver que está en estado `Running`:
```bash
pi@raspberrypi:~ $ kubectl get pods -n kubernetes-what
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          4m30s
```

Tenemos Nginx funcionando en el cluster pero de momento no es posible acceder al mismo desde fuera. Vamos a crear un recurso Service del tipo NodePort:
```bash
pi@raspberrypi:~ $ kubectl expose pod nginx --type=NodePort --port=80 -n kubernetes-what
service/nginx exposed
```

Revisemos en que puerto ha expuesto Kubernetes el servicio (Se podría cambiar pero de momento nos vale):
```bash
pi@raspberrypi:~ $ kubectl get svc -n kubernetes-what
NAME    TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.43.2.65   <none>        80:30279/TCP   3s
```

Por limitaciones de Minikube (Recordemos que funciona en Docker), es necesario el siguiente para exponer un servicio de este tipo:
```
minikube service nginx --url
http://127.0.0.1:57123
```

Si puedes ver la página de bienvenida de Nginx en `http://127.0.0.1:57123` es que todo ha ido bien.

Vamos a dejar todo limpio:
```bash
pi@raspberrypi:~ $ kubectl delete svc -n kubernetes-what nginx
service "nginx" deleted

pi@raspberrypi:~ $ kubectl delete pod -n kubernetes-what nginx
pod "nginx" deleted
```

## Desplegando y exponiendo un Deployment

La idea de este ejercicio es desplegar lo mismos que en el caso anterior pero usando un Deployment y archivos YAML en vez de Pods sueltos y comandos. Empecemos preparando el archivo YAML de nuestro Desployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Para pedir a Kubernetes que levante un recurso de un archivo YAML, el comando es sencillo:
```bash
pi@raspberrypi:~/Downloads $ kubectl apply -f deploy.yaml -n kubernetes-what
deployment.apps/nginx-deployment created
```

Veamos los recursos que hay corriendo en nuestro Namespace:
```bash
pi@raspberrypi:~/Downloads $ kubectl get all -n kubernetes-what
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-7c79c4bf97-gcwzf   1/1     Running   0          14s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   1/1     1            1           14s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-7c79c4bf97   1         1         1       14s
```

Al igual que antes tenemos un Pod pero, como se puede observar, también contamos con un Deployment y un ReplicaSet asociados al mismo. Creemos ahora el Service que falta:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```

En este caso se puede controlar directamente el puerto que expondrá Kubernetes (Rago admitido: 30000-32767). Expongamos el servicio:
```bash
pi@raspberrypi:~/Downloads $ kubectl apply -f service.yaml -n kubernetes-what
service/nginx-svc created

pi@raspberrypi:~/Downloads $ kubectl get services -n kubernetes-what
NAME        TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
nginx-svc   NodePort   10.43.4.21   <none>        80:30007/TCP   14s
```
La página de bienvenida de Nginx debería estar en `http://127.0.0.1:30007`.

Puede ocurrir que tras configurar un Service no puedas acceder al Deployment, una forma de saber que un determinado Service está bien configurado es la siguiente:
```bash
pi@raspberrypi:~/Downloads $ kubectl get pods -n kubernetes-what -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-7c79c4bf97-dj879   1/1     Running   0          12m   10.42.1.12   balthasar   <none>           <none>

pi@raspberrypi:~/Downloads $ kubectl get endpoints -n kubernetes-what
NAME        ENDPOINTS       AGE
nginx-svc   10.42.1.12:80   8m3s
```
Si en los `endpoints` de vuestro servicio no aparece la IP de los Pods desplegados, posiblemente exista algún problema con el selector configurado.

## Editando un recurso

**AVISO: El editor de Kubernetes por defecto es VIM!!**

Algo que permite Kubernetes es el editar algunos de los parámetros de un recurso ya desplegados sin tener que crearlo de nuevo. Aumentemos el número de replicas de nuestro Deployment:
```bash
kubectl edit deployment -n kubernetes-what nginx-deployment

# Se abrirá VIM y podremos editar el YAML del Deployment:
...
spec:
  progressDeadlineSeconds: 600
  replicas: 1 <--- Cambiar a 3
...

deployment.apps/nginx-deployment edited
```

Si todo ha ido bien, dos nuevas replicas de nuestro Deployment deberían aparecer:
```bash
pi@raspberrypi:~/Downloads $ kubectl get pods -n kubernetes-what
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-dj879   1/1     Running   0          25m
nginx-deployment-7c79c4bf97-bbnb8   1/1     Running   0          57s
nginx-deployment-7c79c4bf97-48j9k   1/1     Running   0          57s
```

## Reenvío de puertos 

Antes se ha comentado que existe un tipo de Service que sirve para exponer servicios de forma interna en el cluster, lo que viene genial para practicar con el reenvio de puertos que ofrece la CLI de Kubernetes. En primer lugar borremos el servicio del tipo NodePort anterior, en este caso se muestra como también es posible borrar un recurso a partir del YAML que lo creó:
```bash
pi@raspberrypi:~/Downloads $ kubectl delete -f service.yaml -n kubernetes-what
service "nginx-svc" deleted
```

Preparemos un Service de tipo ClusterIP en un archivo YAML:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Ahora vamos a levantar el Service y nos aseguraremos de que efectivamente funciona como debe:
```bash
pi@raspberrypi:~/Downloads $ kubectl apply -f service.yaml -n kubernetes-what
service/nginx-svc created

pi@raspberrypi:~/Downloads $ kubectl get service -n kubernetes-what
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-svc   ClusterIP   10.43.139.171   <none>        80/TCP    83s

pi@raspberrypi:~/Downloads $ kubectl get pods -n kubernetes-what
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-dj879   1/1     Running   0          35m
nginx-deployment-7c79c4bf97-bbnb8   1/1     Running   0          11m
nginx-deployment-7c79c4bf97-48j9k   1/1     Running   0          11m

pi@raspberrypi:~/Downloads $ kubectl get endpoints -n kubernetes-what
NAME        ENDPOINTS                                   AGE
nginx-svc   10.42.1.12:80,10.42.1.13:80,10.42.2.32:80   101s
```

Bien, para terminar de comprobar que efectivamente tenemos un Nginx que está disponible de forma interna en el cluster, vamos a exponerlo temporalmente mediante el comando `port-forward` de la CLI. El parámetro `--address` puede configurarse con `127.0.0.1` si solo se requiere escuchar en nuestra propia máquina.

```bash
pi@raspberrypi:~/Downloads $ kubectl --namespace kubernetes-what port-forward service/nginx-svc --address 0.0.0.0 30000:80
Forwarding from 0.0.0.0:30000 -> 80
Handling connection for 30000
```

Al acceder al puerto 30000 de nuestra máquina, debería aparecer la página de bienvenida de Nginx: `http://127.0.0.1:30000`.

## Ingress

Exponer un servicio haciendo uso del tipo NodePort del recurso Service no es lo mas conveniente dado que habría que llevar un control de que puertos se encuentran disponibles en los nodos y cuales no. Para evitar este problema, existe la posibilidad de exponer las aplicaciones del cluster mediante un Ingress. Si has llegado hasta aquí deberías tener todo casi preparado para levantar tu primer Ingress, solo falta un detalle.

Dado que un ingress no es más que una regla que se añade a un controlador para determinar como enrutar las diferentes peticiones que llegan en funcion de su domino, deberemos añadir a nuestro archivo `hosts` la siguiente línea:
```
127.0.0.1  kubernetes-what.lan
```
En Windows este archivo se encuentra en la ruta: `C:\Windows\System32\Drivers\etc\hosts` mientras que en Linux lo encontraremos en: `/etc/hosts`.

Creemos ahora el Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: kubernetes-what.lan
    http:
      paths:
      - backend:
          service:
            name: nginx-svc
            port:
              number: 80
        path: /
        pathType: Prefix
```
```bash
pi@raspberrypi:~/Downloads $ kubectl apply -f ingress.yaml -n kubernetes-what
ingress.networking.k8s.io/nginx-ingress created
```

Intenta acceder a `http://kubernetes-what.lan`, el cluster debería enrutarte correctamente hasta los Pods de Nginx.

## Helm

Para acabar, veamos como desplegar un Deployment + Service + Ingress utilizando Helm. Limpiemos todo lo que hemos creado hasta ahora:
```
pi@raspberrypi:~/Downloads $ kubectl delete ingress nginx-ingress -n kubernetes-what
ingress.networking.k8s.io "nginx-ingress" deleted

pi@raspberrypi:~/Downloads $ kubectl delete service nginx-svc -n kubernetes-what
service "nginx-svc" deleted

pi@raspberrypi:~/Downloads $ kubectl delete deployment nginx-deployment -n kubernetes-what
deployment.apps "nginx-deployment" deleted
```

Utilizando la CLI de Helm, despleguemos el Chart que se incluye en este repositorio:

```bash
pi@raspberrypi:~/Downloads/kubernetes-what $ helm install demo-chart demo-chart/ -n kubernetes-what
NAME: demo-chart
LAST DEPLOYED: Sun Mar 31 19:04:19 2024
NAMESPACE: kubernetes-what
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Si se revisa qué recursos se han desplegado en el Namespace `kubernetes-what`, notaremos que tenemos exactamente lo mismo que al final del ejercicio anterior. Prueba a acceder a `http://kubernetes-what.lan`.

Los Charts de Helm ofrecen la posibilidad de personalizar los despliegues mediante los valores que se exponen en el archivo `values.yaml`. Estos valores se pueden cambiar ya sea modificando el archivo directamente o mediante cualquier otro de los métodos disponibles. Para no tener que volver a instalar el Chart vamos a actualizar nuestra instalación actual para aumentar las replicas de los Pods de Nginx, se podría hacer directamente editando el Deployment del cluster pero no es lo ideal:

```
pi@raspberrypi:~/Downloads/kubernetes-what $ kubectl get pods -n kubernetes-what
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-5m7lb   1/1     Running   0          7m22s
```

Para no complicar demasiado el ejercicio, modifiquemos el archivo `values.yaml` del Chart que acabamos de usar:

```yaml
# demo-chart/values.yaml
replicaCount: 3 # Vamos a aumentar las replicas
domain: kubernetes-what.lan

image:
  imageVersion: nginx:latest

```

Actualicemos ahora la instalación del Chart que tenemos en el cluster:
```bash
pi@raspberrypi:~/Downloads/kubernetes-what $ helm upgrade demo-chart demo-chart/ -n kubernetes-what
Release "demo-chart" has been upgraded. Happy Helming!
NAME: demo-chart
LAST DEPLOYED: Sun Mar 31 19:17:45 2024
NAMESPACE: kubernetes-what
STATUS: deployed
REVISION: 2
TEST SUITE: None

pi@raspberrypi:~/Downloads/kubernetes-what $ kubectl get pods -n kubernetes-what
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-5m7lb   1/1     Running   0          13m
nginx-deployment-7c79c4bf97-tfs2f   1/1     Running   0          24s
nginx-deployment-7c79c4bf97-trdch   1/1     Running   0          24s
```

Como se puede ver, Helm facilita la gestión de aplicaciones de un cluster y lo hace más ameno. Antes de acabar, listemos con Helm los Charts instalados y desinstalemos todos los que tengamos en el Namespace `kubernetes-what`.

```bash
pi@raspberrypi:~/Downloads/kubernetes-what $ helm list -n kubernetes-what
NAME            NAMESPACE       REVISION        UPDATED                                         STATUS          CHART                   APP VERSION
demo-chart      kubernetes-what 2               2024-03-31 19:17:45.652379549 +0200 CEST        deployed        demo-chart-1.0.0        1.0.0

pi@raspberrypi:~/Downloads/kubernetes-what $ helm uninstall demo-chart -n kubernetes-what
release "demo-chart" uninstalled
```

Este último comando debería haber limpiado todo lo instalado por Helm, limpiemos ahora el Namespace que nos ha acompañado a lo largo de todos los ejercicios para terminar.

```bash
pi@raspberrypi:~/Downloads/kubernetes-what $ kubectl delete namespace kubernetes-what
namespace "kubernetes-what" deleted
```

**NOTA: ¡Recuerda limpiar también el archivo `hosts`!**
