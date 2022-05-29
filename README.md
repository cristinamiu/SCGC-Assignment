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

