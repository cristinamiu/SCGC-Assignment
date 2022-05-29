# SCGC-Assignment
Let's create a new working directory:
```bash
mkdir assignment
cd assignment
```
## Task 0: Create the Kubernetes cluster

As pointed in the Kubernetes lab, I used the command kind to create the Kubernetes cluster:

```bash
sudo kind create cluster
```

## Task 1: Deploy the nginx service

First let's add the following line to **/etc/hosts** on the host VM in order to give our service a name:

```bash
172.18.0.2 nginx promexporter
```

### 1.1 Create a ConfigMap for the HTML content 

This ConfigMap will display the html for port 80 and 30080.

#### nginx-html-configmap.yaml
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
data:
 index.html: |
    <html>
        <body>
                <h1>Hello from SCGC Assignment!</h1>
                <h3>Miulescu Cristina-Maria, SCPD</h3>
        </body>
    </html>
```

### 1.2 Create a ConfigMap that will take care of the **stub_status** module on port **8080**:

#### nginx-configmap.yaml
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  default.conf: |
    server {
      listen       8080;
      server_name  nginx;


      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
      }

      location /metrics {
        stub_status;
        allow 1.1.1.1;
      }
    }
```

### 1.3 Create the Deployment:
This deployment will have 2 containters with nginx image. The first one will have the customized html file and the second one is for the stub_status metrics.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
      - name: nginx-html
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-html-vol
          mountPath: "/usr/share/nginx/html/index.html"
          subPath: "index.html"
      - name: nginx-metrics
        image: nginx:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: nginx-conf-vol
          mountPath: "/etc/nginx/conf.d/default.conf"
          subPath: "default.conf"
      volumes:
      - name: nginx-conf-vol
        configMap:
          name: nginx-conf
          items:
          - key: "default.conf"
            path: "default.conf"
      - name: nginx-html-vol
        configMap:
          name: nginx-html
          items:
          - key: "index.html"
            path: "index.html"
```

### 1.4 Create the service file:

#### nginx-service.yaml
```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
      name: "html"
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30088
      name: "metrics"
```

### 1.5 Apply the configuration files
```bash
sudo kubectl apply -f nginx-html-configmap.yaml
sudo kubectl apply -f nginx-configmap.yaml
sudo kubectl apply -f nginx-deployment.yaml
sudo kubectl apply -f nginx-service.yaml
```

### Output:

The output can be seen in **/screenshots** folder.

```azure-cli
student@scgc:~/scgc/assignment/task1$ sudo kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-c9bb45656-b5z5z   2/2     Running   0          24s
student@scgc:~/scgc/assignment/task1$ sudo kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                       AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP                       2m21s
nginx        NodePort    10.96.198.183   <none>        80:30080/TCP,8080:30088/TCP   23s
student@scgc:~/scgc/assignment/task1$ curl http://nginx:30080
<html>
    <body>
            <h1>Hello from SCGC Assignment!</h1>
            <h3>Miulescu Cristina-Maria, SCPD</h3>
    </body>
</html>
student@scgc:~/scgc/assignment/task1$ curl http://nginx:30088
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
student@scgc:~/scgc/assignment/task1$ curl http://nginx:30088/metrics
Active connections: 1 
server accepts handled requests
 2 2 2 
Reading: 0 Writing: 1 Waiting: 0 

```

## Task 2: Deploy Prometheus

Next we will go to the **/task2** directory:

```bash
cd task2
```

### 2.1 Create a ConfigMap for promexporter

#### prom-configmap.yaml
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prom-conf
data:
  default.conf: |
    server {
      listen       80;
      server_name  promexporter;


      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
      }
    }
```

### 2.2 Create Deployment for promexporter

#### prom-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: promexporter
  labels:
    app: promexporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: promexporter
  template:
    metadata:
      labels:
        app: promexporter
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: prom-conf-vol
          mountPath: "/etc/nginx/conf.d/default.conf"
          subPath: "default.conf"
      - name: promexporter-container
        image: nginx/nginx-prometheus-exporter:0.8.0
        args: ["-nginx.scrape-uri", "http://nginx:8080/metrics"]
        ports:
        - containerPort: 9113
      volumes:
      - name: prom-conf-vol
        configMap:
          name: prom-conf
          items:
          - key: "default.conf"
            path: "default.conf"
```

### 2.3 Create Service for promexporter

#### prom-service.yaml
```yml
apiVersion: v1
kind: Service
metadata:
  name: promexporter
spec:
  type: NodePort
  selector:
    app: promexporter
  ports:
    - protocol: TCP
      port: 9113
      targetPort: 9113
      nodePort: 30081
      name: "html"
```

### 2.4 Apply the files:
```bash
sudo kubectl apply -f prom-configmap.yaml 
sudo kubectl apply -f prom-deployment.yaml
sudo kubectl apply -f prom-service.yaml
```

### Output:

```azurecli
student@scgc:~/scgc/assignment/task2$ sudo kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
nginx-c9bb45656-b5z5z           2/2     Running   0          27m
promexporter-7796c99ff6-flc5k   2/2     Running   0          5m21s
student@scgc:~/scgc/assignment/task2$ sudo kubectl get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                       AGE
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP                       29m
nginx          NodePort    10.96.198.183   <none>        80:30080/TCP,8080:30088/TCP   27m
promexporter   NodePort    10.96.109.16    <none>        9113:30081/TCP                5m18s
student@scgc:~/scgc/assignment/task2$ curl http://promexporter:30081
<!DOCTYPE html>
			<title>NGINX Exporter</title>
			<h1>NGINX Exporter</h1>
			<p><a href="/metrics">Metrics</a></p>student@scgc:~/scgc/assignment/task2$ 
student@scgc:~/scgc/assignment/task2$ curl http://promexporter:30081/metrics
# HELP nginx_connections_accepted Accepted client connections
# TYPE nginx_connections_accepted counter
nginx_connections_accepted 5
# HELP nginx_connections_active Active client connections
# TYPE nginx_connections_active gauge
nginx_connections_active 1
# HELP nginx_connections_handled Handled client connections
# TYPE nginx_connections_handled counter
nginx_connections_handled 5
# HELP nginx_connections_reading Connections where NGINX is reading the request header
# TYPE nginx_connections_reading gauge
nginx_connections_reading 0
# HELP nginx_connections_waiting Idle client connections
# TYPE nginx_connections_waiting gauge
nginx_connections_waiting 0
# HELP nginx_connections_writing Connections where NGINX is writing the response back to the client
# TYPE nginx_connections_writing gauge
nginx_connections_writing 1
# HELP nginx_http_requests_total Total http requests
# TYPE nginx_http_requests_total counter
nginx_http_requests_total 7
# HELP nginx_up Status of the last metric scrape
# TYPE nginx_up gauge
nginx_up 1
# HELP nginxexporter_build_info Exporter build information
# TYPE nginxexporter_build_info gauge
nginxexporter_build_info{gitCommit="",version=""} 1
student@scgc:~/scgc/assignment/task2$ 

```

