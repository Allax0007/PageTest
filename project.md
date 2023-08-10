---
layout: default
title: Project
description: 'Last edited: 8.8.2023'
---
## Deploying a Database project in Kubernetes 
### Create <i><u>Deployment.yaml</u></i> 
We would deploy two pods: the <code><i>phpapp</i></code> pod contains a container that runs the code we wrote, and the <code><i>database</i></code> pod contains two containers—one running <i>MySQL</i> and the other running <i>phpMyAdmin</i>.
```yaml
# PHPAPP DEPLOY
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
# DATABASE DEPLOY 
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
```
Next, we create three services: <code><i>db-svc</i></code>, <code><i>phpapp-svc</i></code>, and <code><i>phpadmin-svc</i></code>. We specify the <b>spec.clusterIP</b> of <code><i>db-svc</i></code> to allow other services to access it more conveniently.
```yaml
# MySQL SERVICE 
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
# PHPAPP SERVICE 
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
# PhpMyAdmin DEPLOY 
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
Make sure to set the <b>spec.types</b> of all services as <b>NodePort</b> <sub>(default: clusterIP)</sub>, or you will only be able to access them from the local machine.
> The port of the service must be equal to the corresponding container port.

> The Kubernetes control plane allocates a port from a range specified by <code><i>--service-node-port-range</i></code> flag <sub>(default: 30000-32767)</sub>. You can modify the value in <i><u>/etc/kubernetes/manifests/kube-apiserver.yaml</u></i>.

> In YAML, each document or resource definition should be separated by three hyphens (<code>---</code>)  
  You can also write the file separately, depending on your needs.

### Create ConfigMap & Secret
To deploy our deployment more easily, we need to create a <i>ConfigMap</i>.  
In the <i>ConfigMap</i>, set <code>test</code> as the database we use, the specified <b>clusterIP</b> as <code>PMA_HOST</code>, and the specified <b>port</b> of <code>db-svc</code> as <code>PMA_PORT</code>.
```shell
kubectl create cm db-config --from-literal=MYSQL_DATABASE=test
kubectl create cm phpadmin-config --from-literal=PMA_HOST=10.107.0.1 --from-literal=PMA_PORT=3306
```
Lastly, create a secret to set your <code>MYSQL_ROOT_PASSWORD</code>
```shell
kubectl create secret generic db-secret --from-literal=MYSQL_ROOT_PASSWORD=<password>
```
### What <code><i>phpapp</i></code> Do
In the <code><i>phpapp</i></code> pod, we build a code that queries <i>MySQL</i> in the <code><i>database</i></code> pod and performs <code>INSERT</code> <code>UPDATE</code> operations with the <code><i>test</i></code> database.

## Set Up a Prometheus Monitoring Environment Using Helm Chart 
### Install Helm 
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm
```
### Initialize a Helm Chart Repository 
Add repository to Helm and update local repositories.
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
In the following steps, we will install <code><i>kube-prometheus-stack</i></code>, which collects <code>Kubernetes manifests</code>, <code>Grafana dashboards</code>, and <code>Prometheus rules</code> combined with documentation and scripts.
### Install Prometheus 
```shell
helm install prom prometheus-community/kube-prometheus-stack -n monitor --create-namespace
```
It will install <code><i>kube-prometheus-stack</i></code> chart in a namespace called <code><i>monitor</i></code>.

## Sending Alertmanager Alerts to Discord Server Using Webhook 
### Clone <i><u>values.yaml</u></i> 
To efficiently configure the repository, we need to copy a file to the current directory.
```shell
helm show values prometheus-community/kube-prometheus-stack > values.yaml
```

### Modify the Configs in <i>values.yaml</i> 
#### additionalPrometheusRulesMap <sub>[line 164]</sub>  
You can apply your rule here.  
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
#### alertmanager.config <sub>[line 249]</sub>
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
        text: '{{ template "slack.myorg.text" . }}'
        send_resolved: true
    templates:
    - '/etc/alertmanager/config/*.tmpl'
```
Replace <code>api_url</code> with the Webhook URL of your Discord server.

If you want to receive notifications only for <i>FIRING</i> cases, you can remove <code>send_resolved: true</code> <sub>(default: false)</sub>
#### alertmanager.templateFiles <sub>[line 313]</sub> 
Uncomment the lines in the provided template. Alternatively, you can create your template here.
```yaml
  templateFiles: 
  #
  ## An example template:
    # Uncomment starting from this line...
```
<mark>If you want to access Prometheus from external IP, you should make modifications as below.</mark>
#### alertmanager.service.type <sub>[line 464]</sub> (optional)
```yaml
  ## Configuration for Alertmanager service
  ##
  service:

    ## ...
    ## Values unchanged... 
    ## ...

    type: NodePort
```
#### prometheus.service.type <sub>[line 2574]</sub> (optional)
```yaml
  ## Configuration for Prometheus service
  ##
  service:

    ## ...
    ## Values unchanged... 
    ## ...

    type: NodePort
```
You can also modify <i>service.type</i> after applying changes.
```shell
kubectl patch svc svcName -n monitor -p '{"spec": {"type": "NodePort"}}'
```
> Changing the type of the service using <code>kubectl patch</code> may result in the NodePort not being the value set in <i><u>values.yaml</u></i>
#### grafana.enabled <sub>[line 857]</sub> (optional)
To save the resources, if Grafana is not needed, it's recommended to set it to false.
### Applying the Changes 
```shell
helm upgrade prom -f values.yaml prometheus-community/kube-prometheus-stack -n monitor
```
After making the modifications, you can go to [Prometheus Server](http://192.168.158.66:30090) to check if the custom rule has been applied. 

## Monitoring the Status of Nodes with Python
First, make sure you have the <code>requests</code> and <code>matplotlib</code> libraries installed. If not, you can install them using <code>pip install requests matplotlib</code>.

To get memory usage of each pod from Prometheus every second and draw a dynamic graph, you can use the <code>matplotlib</code> library in Python to create the graph. Additionally, you can use the <code>FuncAnimation</code> class from <code>matplotlib.animation</code> to continuously update the graph with new data at regular intervals.

Here is an example to make a dynamic line chart, which marks the CPU usage and memory usage of containers.

Use a global variable <code>historical_data</code> to store historical data. When updating the chart, we save the new memory usage data into this dictionary.
```python3
historical_data = {'cpu': {}, 'memory': {}}
```
Set the values for the API and query (<i>promQL</i>).
```python3
    url = 'http://prometheus_server_ip:30090/api/v1/query'
    if metric == 'cpu':
        query = 'sum by(pod) (node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace!="kube-system"})'
    elif metric == 'memory':
        query = 'sum(container_memory_working_set_bytes{job="kubelet", metrics_path="/metrics/cadvisor", namespace!="kube-system"}) by (pod)'
    else:
        return None

    params = {'query': query}
```

Replace <code>prometheus_server_ip</code> with the IP address or domain name of your Prometheus server and <code>your_namespace</code> with the desired Kubernetes namespace containing the pods you want to monitor.

Request Prometheus to get data.
```python3
    try:
        response = requests.get(url, params=params)
        response.raise_for_status()
        data = response.json()

        if data['status'] == 'success':
            return data['data']['result']
        else:
            print('Error:', data['error'])
            return None

    except requests.exceptions.RequestException as e:
        print('Error:', e)
        return None
```
Sort the data by its pod name and append to <code>historical_data</code>
```python3
        # Update CPU Usage
        for pod_data in pod_cpu_usage:
            pod_name = pod_data['metric']['pod']
            cpu_usage_rate = float(pod_data['value'][1])

            if pod_name not in historical_data['cpu']:
                historical_data['cpu'][pod_name] = {'time': [], 'usage': []}

            historical_data['cpu'][pod_name]['time'].append(current_time)
            historical_data['cpu'][pod_name]['usage'].append(cpu_usage_rate)
        # Update memory Usage
        for pod_data in pod_memory_usage:
            pod_name = pod_data['metric']['pod']
            memory_usage_bytes = float(pod_data['value'][1])
            memory_usage_mb = memory_usage_bytes / (1024 ** 2)

            if pod_name not in historical_data['memory']:
                historical_data['memory'][pod_name] = {'time': [], 'usage': []}

            historical_data['memory'][pod_name]['time'].append(current_time)
            historical_data['memory'][pod_name]['usage'].append(memory_usage_mb)
```
Create the figure.
```python3
        plt.clf()
        # CPU Usage Figure
        plt.subplot(2, 1, 1)
        plt.title("CPU Usage of Each Pod Over Time")
        for pod_name, data in historical_data['cpu'].items():
            time_data = data['time']
            cpu_data = data['usage']
            plt.plot(time_data, cpu_data, linewidth=3, label=f"{pod_name} ({cpu_data[-1]:.4f})")

        plt.xlabel('Time (seconds)')
        plt.ylabel('CPU Usage')
        plt.legend(loc='center left')
        plt.xticks(rotation=45, ha='right')
        plt.grid(True) 

        # Memory Usage Figure
        plt.subplot(2, 1, 2)
        plt.title("Memory Usage of Each Pod Over Time", y=-0.15)
        for pod_name, data in historical_data['memory'].items():
            time_data = data['time']
            memory_data = data['usage']
            plt.plot(time_data, memory_data, linewidth=3, label=f"{pod_name} ({memory_data[-1]:.2f} MB)")

        plt.ylabel('Memory Usage (MB)')
        plt.legend(loc='center left')
        plt.xticks(rotation=45, ha='right', visible=False)
        plt.grid(True) 
        plt.tight_layout() 
```
In order to ensure the successful update of the plots, it is necessary to use <code>clf()</code> to clear the current figure.   
Finally, output the figure and update it every 100 ms.
```python3
if __name__ == "__main__":
    fig = plt.figure(figsize=(10, 6))
    ani = FuncAnimation(fig, update_graph, interval=100)
    plt.show()
```
<code>figsize</code> is used to set the figure size, a is the width of the figure, b is the height of the figure <sub>(unit: inch)</sub>.

## Troubleshooting 
At the beginning, you may encounter many errors. The following will analyze and solve them one by one.

### HighDiskPressure 
In the terminal, enter <code>df -h</code> to view disk usage. If the capacity is insufficient, please clean up unused files to free up space.  
If the capacity is sufficient and kubelet continues to report errors, you can go to the kubelet configuration file and increase the value of <b>HighThresholdPercent</b> <sub>(default:90)</sub> to avoid unnecessary pod evictions.  
If the issue persists, please contact your device provider.

### Alertmanager - Many Alerts Firing 
#### AlertmanagerClusterFailedToSendAlerts 
If your <code>api_url</code> or <b>alertmanager.templateFiles</b> is invalid, it will be in a FIRING state.

* Please check whether your <code>api_url</code> has any issues. Otherwise, your <code>api_url</code> should be able to receive messages in your Discord server.

* Please refrain from altering the <b>alertmanager.templateFiles</b> unless you are familiar with handling it. It is recommended to keep it as the default setting.


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
