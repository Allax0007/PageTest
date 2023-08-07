---
layout: default
title: Project
description: Act. 2023
---
## Deploying a Database project in Kubernetes 
### Create Deployment.yaml 
We would deploy two pods: the <code>phpapp</code> pod contains a container that runs the code we wrote, and the <code>database</code> pod contains two containers—one running MySQL and the other running phpMyAdmin.

Next, we create three services: <code>db-svc</code>, <code>phpapp-svc</code>, and <code>phpadmin-svc</code>. We specify the clusterIP of <code>db-svc</code> to allow other services to access it more conveniently.

Lastly, make sure to set the types of all services as <b>NodePort</b> (default: clusterIP), or you will only be able to access them from the local machine.
```yaml
/* PHPAPP DEPLOY */
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpapp
spec:
  replicas: 1
  selector:
    matchLabels:
      run: phpapp
  template:
    metadata:
      name: phpapp-deploy
      labels:
        run: phpapp
    spec:
      containers:
      - name: phpapp
        image: allax0007/dbphp
        ports:
        - containerPort: 8000
---
/* DATABASE DEPLOY */
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 1
  selector:
    matchLabels:
      run: db
  template:
    metadata:
      name: db-deploy
      labels:
        run: db
    spec:
      containers:
      - name: db
        image: mysql
        envFrom:
        - configMapRef:
            name: db-config
        - secretRef:
            name: db-secret
        ports:
        - containerPort: 3306
      - name: phpadmin
        image: phpmyadmin
        envFrom:
        - configMapRef:
            name: phpadmin-config
        ports:
        - containerPort: 7000
---
/* MySQL SERVICE */
apiVersion: v1
kind: Service
metadata:
  labels:
    run: db
  name: db-svc
spec:
  clusterIP: 10.107.0.1
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    run: db
  type: ClusterIP
---
/* PHPAPP SERVICE */
apiVersion: v1
kind: Service
metadata:
  labels:
    run: phpapp
  name: phpapp-svc
spec:
  ports:
  - nodePort: 30002
    port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    run: phpapp
  type: NodePort
---
/* PhpMyAdmin DEPLOY */
apiVersion: v1
kind: Service
metadata:
  labels:
    run: db
  name: phpadmin-svc
spec:
  ports:
  - nodePort: 30001
    port: 7000
    protocol: TCP
    targetPort: 80
  selector:
    run: db
  type: NodePort
```

> NOTE: In YAML, each document or resource definition should be separated by three hyphens (<code>---</code>)  
  You can also write the file separately, depending on your needs.

### Create ConfigMap & Secret
To deploy our deployment more easily, we need to create a ConfigMap.  
In the ConfigMap, set <code>test</code> as the database we use, the <i>specified clusterIP</i> as <code>PMA_HOST</code>, and the <i>specified port of db-svc</i> as <code>PMA_PORT</code>.
```shell
kubectl create cm db-config --from-literal=MYSQL_DATABASE=test
kubectl create cm phpadmin-config --from-literal=PMA_HOST=10.107.0.1 --from-literal=PMA_PORT=3306
```
Lastly, create a secret to set your <code>MYSQL_ROOT_PASSWORD</code>
```shell
kubectl create secret generic db-secret --from-literal=MYSQL_ROOT_PASSWORD=<password>
```
### What <code>phpapp</code> Do
In the <code>phpapp</code> pod, we build a code that queries MySQL in the <code>database</code> pod and performs <code>INSERT</code> <code>UPDATE</code> operations with the <code>test</code> database.

## Set Up a Prometheus Monitoring Environment Using Helm Chart 
### Install Helm 
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm
```
### Initialize a Helm Chart Repository 
Add repository to Helm as below:
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
### Install Prometheus 
```shell
helm install prom prometheus-community/kube-prometheus-stack -n monitor --create-namespace
```
It will install <code>kube-prometheus-stack</code> chart in a namespace called <code>monitor</code>.

## Sending Alertmanager Alerts to Discord Server Using Webhook 
### Clone <i>values.yaml</i> 
To efficiently configure the repository, we need to copy a file to the current directory.
```shell
helm show values prometheus-community/kube-prometheus-stack --version 48.2.1 > values.yaml
```

### Modify the Configs in <i>values.yaml</i> 
* <b>additionalPrometheusRulesMap</b> <sub>[line 164]</sub>  
You can apply your own rule here.  
e.g., The following rules will trigger an alert when the corresponding events occur.
```yaml
additionalPrometheusRulesMap:
  rule-name:
    groups:
      - name: Alex-rules
        rules:
          - alert: HighDiskPressure
            expr: node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} < 0.1
            for: 1m
            labels:
              severity: warning
            annotations:
              summary: "High disk pressure on {{ $labels.instance }}"
              description: "Disk pressure is high on {{ $labels.instance }}"
          - alert: HighMemoryUsage
            expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.2
            for: 1m
            labels:
              severity: warning
            annotations:
              summary: "High memory usage on {{ $labels.instance }}"
              description: "Memory usage is high on {{ $labels.instance }}"
         - alert: HighCPULoad
           expr: (sum by (instance) (avg by (mode, instance) (rate(node_cpu_seconds_total{mode!="idle"}[2m]))) > 0.8) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
           for: 1m
           labels:
             severity: warning
           annotations:
             summary: High CPU load (instance {{ $labels.instance }})
             description: "CPU load is > 80%\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```
* <b>alertmanager.config</b> <sub>[line 249]</sub>
Here is an example of sending an alert to the Discord server.
```yaml
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 48h
      # Default alerter
      receiver: 'null'
      routes:
      # "Muted" alerts, currently only false positive "Watchdog" resides here
      - receiver: "null"
        matchers:
        - alertname =~ "Watchdog|InfoInhibitor"
      # Nosiy alerts
      - receiver: "null"
        matchers:
        - alertname =~ "UnusedCpu|UnusedMemory"

      # Matches anything
      - receiver: "discord"

    receivers:
    - name: "null"

    - name: "discord"
      slack_configs:
      - channel: 'alerts'
        username: "Alert"
        api_url: 'https://discord.com/api/webhooks/*******************/************************************************/slack'

    templates:
    - '/etc/alertmanager/config/*.tmpl'
```
※ <b>If you want to access Prometheus from another IP, you should make the following modifications.</b>

* <b>alertmanager.service.type</b> <sub>[line 464]</sub> (optional)
```yaml
  ## Configuration for Alertmanager service
  ##
  service:

    ## ...
    ## Values unchanged... 
    ## ...

    type: NodePort
```

* <b>prometheus.service.type</b> <sub>[line 2574]</sub> (optional)
```yaml
  ## Configuration for Prometheus service
  ##
  service:

    ## ...
    ## Values unchanged... 
    ## ...

    type: NodePort
```
You can also modify service.type after applying changes.
```shell
kubectl patch svc svcName -n monitor -p '{"spec": {"type": "NodePort"}}'
```
> <b>NOTE: Changing the type of the service using <code>kubectl patch</code> may result in the NodePort not being the value set in values.yaml</b>

### Applying the Changes 
```shell
helm upgrade prom -f values.yaml prometheus-community/kube-prometheus-stack -n monitor
```
After making the modifications, you can go to [Prometheus Server](http://192.168.158.66:30090) to check if the custom rule has been applied. 

If <i>AlertmanagerClusterFailedToSendAlerts</i> is inactive, you should be able to receive messages in your Discord server.

## Troubleshooting 
At the beginning, you may encounter many errors. The following will analyze and solve them one by one.

### HighDiskPressure 
In the terminal, enter <code>df -h</code> to view disk usage. If the capacity is insufficient, please clean up unused files to free up space.  
If the capacity is sufficient and kubelet continues to report errors, you can go to the kubelet configuration file and increase the value of <b>HighThresholdPercent</b> <sub>(default:90)</sub> to avoid unnecessary pod evictions.  
If the issue persists, please contact your device provider.

### Alertmanager - Many Alerts Firing 

#### Watchdog 
This is an alert that should always be firing to certify that Alertmanager is working properly. Hence, we have nothing to do with it. 

#### TargetDown 
This alert is influenced by other alerts. If any alert with a name containing 'Down' is triggered, then this alert is also triggered.

#### etcdMembersDown 
If the Prometheus Server Console shows <code>Get "192.168.***.***:2381/metrics": dial tcp <yourIP>:2381: connect: connection reset.</code> 

Run the following command:
```shell
cat /etc/kubernetes/manifests/etcd.yaml  | egrep -i '\--listen-metrics'
```
If <code>- --listen-metrics-urls=<nowiki>http://127.0.0.1:2381</nowiki></code> is displayed, please modify the argument to <code>- --listen-metrics-urls=http://0.0.0.0<nowiki/>:2381</code>

The change will take effect immediately.
#### etcdInsufficientMembers 
Once the problem above is fixed, this alert will become inactive.  

#### KubeProxyDown 
If the Prometheus Server Console shows<code>Get "prometheus_server_ip:10249/metrics": dial tcp prometheus_server_ip:10249: connect: connection refused.</code>

Set the kube-proxy argument for metric-bind-address as below.
```
$ kubectl edit cm/kube-proxy -n kube-system

...
kind: KubeProxyConfiguration
metricsBindAddress: 0.0.0.0:10249
...

$ kubectl delete pod -l k8s-app=kube-proxy -n kube-system
```
> <b>NOTE: You must delete the existing pods to apply the changes.</b>
#### KubeControllerManagerDown
If the Prometheus Server Console shows <code>Get "prometheus_server_ip:10257/metrics": dial tcp prometheus_server_ip:10257: connect: connection refused.</code> 

Run the following command:
```shell
cat /etc/kubernetes/manifests/kube-controller-manager.yaml  | egrep -i '\--bind'
```
If only <code>- --bind-address=127.0.0.1</code> is displayed, please add <code>- --bind-address=0.0.0.0</code> to the relevant file.
The change will take effect immediately.

#### KubeSchedulerDown 
If the Prometheus Server Console shows: <code>Get "prometheus_server_ip:10259/metrics": dial tcp prometheus_server_ip:10259: connect: connection refused.</code>

Just follow the steps above, change the target file to <u>kube-scheduler.yaml</u>, make the necessary changes, and save the file.
