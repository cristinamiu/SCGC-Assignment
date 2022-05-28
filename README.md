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
172.18.0.2 nginx
```

### 1.1 Create the configuration map for index.html
This ConfigMap is used to display the index.html on port 80 within the cluster and port 30080 outside the cluster.

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

### 1.2 Create the default configuration map:
In this ConfigMap we will add along with the index.html, the **stub_status** for the **/metrics** path module.

#### nginx-configmap
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  default.conf: |
    server {
      listen       80;
      server_name  localhost;


      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
      }

      location /metrics {
        stub_status;
        allow 127.0.0.1;
        deny all;
      }
    }

```

### 1.3 Create the deployment:

#### nginx-deployment.yaml

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
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-html-vol
          mountPath: "/usr/share/nginx/html/index.html"
          subPath: "index.html"
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

### 1.4 Create the service:

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
      targetPort: 80
      nodePort: 30088
      name: "metrics"
```

### Output:


```bash
student@scgc:~/scgc/assignment$ sudo curl http://nginx:30080
<html>
    <body>
            <h1>Hello from SCGC Assignment!</h1>
            <h3>Miulescu Cristina-Maria, SCPD</h3>
    </body>
</html>
student@scgc:~/scgc/assignment$ sudo curl http://nginx:30088
<html>
    <body>
            <h1>Hello from SCGC Assignment!</h1>
            <h3>Miulescu Cristina-Maria, SCPD</h3>
    </body>
</html>
student@scgc:~/scgc/assignment$ sudo curl http://nginx:30088/metrics
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.21.6</center>
</body>
</html>
```
For now the /metrics path does not show us the desired result.
Next, we will use the prometheus-exporter.

## Task 2: Deploy a prometheus exporter

For this task, I have added a new container with the port 9113.
The nginx-deployment file looks like this:

#### nginx-deployment.yaml
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
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-html-vol
          mountPath: "/usr/share/nginx/html/index.html"
          subPath: "index.html"
        - name: nginx-conf-vol
          mountPath: "/etc/nginx/conf.d/default.conf"
          subPath: "default.conf"
      - name: adapter
        image: nginx/nginx-prometheus-exporter:0.8.0
        args: ["-nginx.scrape-uri", "http://localhost/metrics"]
        ports:
        - containerPort: 9113
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
Also, the nginx-service.yaml has been modified so that for port 8080 we will have as the targetPort the new prometheus container:

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
      targetPort: 9113
      nodePort: 30088
      name: "metrics"

```

Output:
```bash
student@scgc:~/scgc/assignment$ curl http://nginx:30080
<html>
    <body>
            <h1>Hello from SCGC Assignment!</h1>
            <h3>Miulescu Cristina-Maria, SCPD</h3>
    </body>
</html>
student@scgc:~/scgc/assignment$ curl http://nginx:30088
<!DOCTYPE html>
			<title>NGINX Exporter</title>
			<h1>NGINX Exporter</h1>
			<p><a href="/metrics">Metrics</a></p>student@scgc:~/scgc/assignment$ 
student@scgc:~/scgc/assignment$ curl http://nginx:30088/metrics
# HELP nginx_connections_accepted Accepted client connections
# TYPE nginx_connections_accepted counter
nginx_connections_accepted 6
# HELP nginx_connections_active Active client connections
# TYPE nginx_connections_active gauge
nginx_connections_active 1
# HELP nginx_connections_handled Handled client connections
# TYPE nginx_connections_handled counter
nginx_connections_handled 6
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
student@scgc:~/scgc/assignment$ 

```
