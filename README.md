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

<img src="./screenshots/Task0%20-%20create%20cluster.png"/>

## Task 1: Deploy the nginx service

First let's add the following line to **/etc/hosts** on the host VM in order to give our service a name:

```bash
172.18.0.2 nginx promexporter
```

Create **task1** directory:
```bash
mkdir task1
cd task1
```

### 1.1 Create a ConfigMap for the HTML content 

This ConfigMap will display the html for port **80** within the cluster and port  **30080** outside the cluster.

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

The terminal output looks as below:

<img src="./screenshots/Task1%20-%20output%20terminal.png"/>

Also, in the browser:
- left side: http://nginx:30088/metrics
- right side: http://nginx:30080

<img src="./screenshots/Task1%20-%20output%20browser.png"/>

## Task 2: Deploy Prometheus

Next we will go to the **/task2** directory:

```bash
mkdir task2
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
The promexporter will get its metrics info from http://nginx:8080/metrics :
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

The promexporter will have port **9113** within the cluster and port **30081** outside the cluster:
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

The terminal output:

<img src="./screenshots/Task2%20-%20output%20terminal.png"/>

The browser output:
- left side: http://promexporter:30081/metrics
- right side: http://promexporter:30081

<img src="./screenshots/Task2%20-%20output%20browser.png">

## Task 3: 

### 3.1 Create **monitoring** namespace:

```bash
sudo kubectl create namespace monitoring
```

### 3.2 Install prometheus using helm
```bash
$ sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ sudo helm install prometheus prometheus-community/prometheus -n monitoring
```

### 3.3 Port-forrward

```bash
sudo kubectl -n monitoring port-forward services/prometheus-server 30082:80
```

### 3.4 Open a new terminal and edit the configMap of prometheus-server:
```bash
sudo kubectl -n monitoring edit cm prometheus-server
```

Add this line:
```yml
- job_name: prometheus
      static_configs:
      - targets:
        - localhost:9090
    # Add this
    - job_name: promexporter
      static_configs:
      - targets:
        - promexporter:30081
```
**Note:** I also tried with _promexporter.default.svc.cluster.local:9113_ as indicated but it did not show the metrics when accessing the link.

### 3.5 Test

```bash
curl http://localhost:30082
```

### Output:

Going to http://localhost:30082, you shold see the following:

<img src="./screenshots/Task3%20-%20view.png">

Under **/targets** :

<img src="./screenshots/Task3%20-%20output%20browser.png">

We can find under **/targets** our **promexporter**:

<img src="./screenshots/Task3%20-%20output%20promexporter.png">

<br/>

#### Query a metric:

Also, go to **Graph** and query a metric, like “nginx_connections_accepted”:

<img src="./screenshots/Task3%20-%20query.png">

## Task 4: Grafana

### 4.1 Install Grafana
Next, I installed Grafana using helm chart on the **monitoring** namespace:

```bash
sudo helm repo add bitnami https://charts.bitnami.com/bitnami
sudo helm install grafana bitnami/grafana
```

<img src="./screenshots/Task4%20-%20grafana.png">


### 4.2 Port-forward Grafana

```bash
sudo kubectl -n monitoring port-forward svc/grafana 30085:3000
```

### 4.3 Grafana UI

We can access Grafana UI on http://localhost:30085, the port we forwarded to.

First we have to find out the password in order to login:

```bash
student@scgc:~$ echo "Password: $(sudo kubectl get secret grafana-admin --namespace monitoring -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 --decode)"
[sudo] password for student: 
Password: 2kOvyXZklL
```

So the credentials are:
- Username: admin
- Password: 2kOvyXZklL

<img src="./screenshots/Task4%20-%20grafana%20ui.png">


### 4.4 Create Prometheus DataSource

The only field modified is the URL field: http://prometheus-server .

<img src="./screenshots/Task4%20-%20datasrc.png">

<img src="./screenshots/Task4%20-%20datasource.png">

### 4.5 Create Prometheus Dashboard:

I copied the content from https://github.com/nginxinc/nginx-prometheus-exporter/blob/main/grafana/dashboard.json .

<img src="./screenshots/Task4%20-%20import%20dashboard.png">

<img src="./screenshots/Task4%20-%20dasboard.png">

### Output

<img src="./screenshots/Task4%20-%20graphs%20connections.png">

<img src="./screenshots/Task4%20-%20graphs%20connections2.png">