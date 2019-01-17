
# Lab 4 - Exploring Mixed Workloads in a kubernetes cluster

## Part 1 - Creating the cluster

Prereqs:

- Azure subscription
- install [azure-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- install [helm](https://docs.helm.sh/using_helm/)
- install [aks-engine](https://github.com/Azure/aks-engine/releases/latest)
- ssh keys

### Creating the cluster with aks-engine

Aks-engine will allow us to create our kubernetes cluster with windows nodes. Before you follow this [walkthrough](https://github.com/Azure/aks-engine/blob/077f396b69fd5674fc05e130fe8780045f136d9d/docs/topics/windows.md), make note that we will want to use the [windows/kubernetes-hybrid.json](https://github.com/Azure/aks-engine/blob/077f396b69fd5674fc05e130fe8780045f136d9d/examples/windows/kubernetes-hybrid.json) apimodel to deploy a cluster with 2 Windows nodes, and 2 Linux nodes (5 total nodes, including the master) instead of the simple cluster with 2 Windows nodes.

[Learn more about Windows and Kubernetes](https://github.com/Azure/aks-engine/blob/077f396b69fd5674fc05e130fe8780045f136d9d/docs/topics/windows-and-kubernetes.md)

## Part 2 - Logging and monitoring

### Logging

Now that you have a mixed-cluster up with a running service, let's look at logging. There are a few [considerations](https://gist.github.com/jsturtevant/73b0bfe301a6abecd951b6f98bddffd4) when thinking about logging options for windows workloads on kubernetes.

Similar to previous labs in this workshop, windows applications often maintain logs in files. In this lab, we will leverage [FluentD](https://github.com/fluent/fluentd) in order to observe these logs in a way that is more native to kubernetes, i.e via the ```kubectl logs``` command.

#### FluentD Configuraton

As it currently stands, Docker Automated builds do not support windows. For this reason, we will need to manually build and push to our own repo. In order to do so, you need to:

1. Get the latest FluentD Dockerfile for Windows. As of Jan 7, 2019, [v1.3](https://raw.githubusercontent.com/fluent/fluentd-docker-image/master/v1.3/windows/Dockerfile) isthe latest.
1. Create a fluent.conf file in the same directory.

To configure FluentD to gather events from a file, we'll configure the ```tail``` input plugin to watch a directory for .log files, then to configure FluentD to output those events to stdout, we'll configure the ```stdout``` output plugin. The contents of your fluent.conf should look something like this:

```conf
# configure input plugins using source directive
# NOTE: the path string must use '/' instead of '\'
<source>
  @type tail
  format none
  path C:/logs/*.log
  pos_file C:/fluentd_test.pos
  tag iis.*
  rotate_wait 5
  read_from_head true
  refresh_interval 60
</source>

# configure output plugins using match directive
<match iis.**>
  @type stdout
</match>
```

The base images of your app and the fluentd sidecar, need to be compatible. In this case, the IIS image is using windowsservercore-ltsc2019. Let's make this change in our Dockerfile:

```Dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2019
...
```

Now you're ready to build and push your image:

```sh
# build locally (this could take a few minutes) and push to repo
docker build -t {repo}/fluentd-win:1.3 .
docker push {repo}/fluentd-win:1.3
```

#### Kubernetes deployment

There are two main differences in our deployment:

1. the addition of the fluentd container
1. a shared volume

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iis-with-logging
  labels:
    app: iis-with-logging
spec:
  replicas: 1
  template:
    metadata:
      name: iis-with-logging
      labels:
        app: iis-with-logging
    spec:
      containers:
      - name: iis
        image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
        resources:
          limits:
            cpu: 1
            memory: 800m
          requests:
            cpu: .1
            memory: 300m
        ports:
          - containerPort: 80
        volumeMounts:
        - mountPath: C:\inetpub\logs\LogFiles\w3svc1
          name: log-volume
      - name: fluentd
        image: {repo}/fluentd-win:1.3
        imagePullPolicy: Always
        volumeMounts:
        - mountPath: C:\logs
          name: log-volume
      volumes:
      - name: log-volume
        emptyDir: {}
      nodeSelector:
        "beta.kubernetes.io/os": windows
  selector:
    matchLabels:
      app: iis-with-logging
---
apiVersion: v1
kind: Service
metadata:
  name: iis-with-logging
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
  selector:
    app: iis-with-logging
```

Now you should be able to run ```kubectl logs -lapp=iis-with-logging -c fluentd``` to view iis events.

Example output:

```sh
kubectl logs -lapp=iis-1803-with-logging -c fluentd
2019-01-07 20:25:11 +0000 [info]: parsing config file is succeeded path="C:\\fluent\\conf\\fluent.conf"
2019-01-07 20:25:13 +0000 [info]: using configuration file: <ROOT>
  <match fluent.**>
    @type null
  </match>
  <source>
    @type tail
    format none
    path "C:/logs/*.log"
    pos_file "C:/fluentd_test.pos"
    tag "iis.*"
    rotate_wait 5
    read_from_head true
    refresh_interval 60
    <parse>
      @type none
    </parse>
  </source>
  <match iis.**>
    @type stdout
  </match>
</ROOT>
2019-01-07 20:25:13 +0000 [info]: starting fluentd-1.3.2 pid=20684 ruby="2.4.2"
2019-01-07 20:25:13 +0000 [info]: spawn command to main:  cmdline=["C:/ruby24/bin/ruby.exe", "-Eascii-8bit:ascii-8bit", "C:/ruby24/bin/fluentd", "-c", "C:\\fluent\\conf\\fluent.conf", "--under-supervisor"]
2019-01-07 20:25:18 +0000 [info]: gem 'fluentd' version '1.3.2'
2019-01-07 20:25:18 +0000 [info]: adding match pattern="fluent.**" type="null"
2019-01-07 20:25:18 +0000 [info]: adding match pattern="iis.**" type="stdout"
2019-01-07 20:25:18 +0000 [info]: adding source type="tail"
2019-01-07 20:25:18 +0000 [info]: #0 starting fluentd worker pid=19804 ppid=20684 worker=0
2019-01-07 20:25:18 +0000 [info]: #0 fluentd worker is now running worker=0
2019-01-07 20:26:18 +0000 [info]: #0 following tail of C:/logs/u_ex190107.log
2019-01-07 20:26:39.561234000 +0000 iis.C:.logs.u_ex190107.log: {"message":"#Software: Microsoft Internet Information Services 10.0"}
2019-01-07 20:26:39.561234000 +0000 iis.C:.logs.u_ex190107.log: {"message":"#Version: 1.0"}
2019-01-07 20:26:39.561234000 +0000 iis.C:.logs.u_ex190107.log: {"message":"#Date: 2019-01-07 20:26:01"}
2019-01-07 20:26:39.561234000 +0000 iis.C:.logs.u_ex190107.log: {"message":"#Fields: date time s-ip cs-method cs-uri-stem cs-uri-query s-port cs-username c-ip cs(User-Agent) cs(Referer) sc-status sc-substatus sc-win32-status time-taken"}
2019-01-07 20:26:39.561234000 +0000 iis.C:.logs.u_ex190107.log: {"message":"2019-01-07 20:26:01 10.240.0.85 GET / - 80 - 10.240.0.65 Mozilla/5.0+(Windows+NT+10.0;+Win64;+x64)+AppleWebKit/537.36+(KHTML,+like+Gecko)+Chrome/64.0.3282.140+Safari/537.36+Edge/18.17763 - 200 0 0 714"}
2019-01-07 20:26:39.561234000 +0000 iis.C:.logs.u_ex190107.log: {"message":"2019-01-07 20:26:01 10.240.0.85 GET /iisstart.png - 80 - 10.240.0.65 Mozilla/5.0+(Windows+NT+10.0;+Win64;+x64)+AppleWebKit/537.36+(KHTML,+like+Gecko)+Chrome/64.0.3282.140+Safari/537.36+Edge/18.17763 http://40.85.160.140/ 200 0 0 287"}
```

### Monitoring

It is important to observe and monitor the health of your cluster. There are a few options available for monitoring Kubernetes cluster, some of the main ones are highlighted [here](https://github.com/Azure/aks-engine/blob/077f396b69fd5674fc05e130fe8780045f136d9d/docs/topics/monitoring.md). In this part of the lab, we will be looking at Azure Monitor (formerly Operations Management Suite) as well as how Windows Management Instrumentation can be leveraged to export metrics to Prometheus.

#### Container Insights and Log Analytics 

Azure Monitor allows you to monitor, analyze, and visualize the health of all your Azure applications and services whereever they are hosted in one location. You can learn more about Azure Monitor for Containers [here](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/container-insights-overview).

Follow the steps [here](https://github.com/Microsoft/OMS-docker/tree/aks-engine) to add Azure Monitoring for Containers to your cluster.
> Tip: Leverage the [AddAzureMonitor-Containers.ps1](https://raw.githubusercontent.com/vyta/windows-containers-workshop/kubernetes/labs/lab-04/monitoring/AddAzureMonitor-Containers.ps1) when adding the Container Insights Solution to your workspace.

Now you should see data from your linux nodes. There currently isn't a containerized omsagent for windows, so there is additional work needed to get those nodes up to speed:

[Installing OMS Agent on Master Node](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/azure-monitor/insights/containers.md#install-and-configure-windows-container-hosts):

  1. First we need to prep the windows nodes before installing the agent:

    - RDP into each Windows Node:

    ```bash
    ssh -L 5500:<node-ip>:3389 linuxuser@<clustername>.<region>.cloudapp.azure.com

    # Launch RDP to connect to localhost:5500 and use Windows credentials to login
    ```
    - Then run the prep script:

    ```bash
    powershell Invoke-WebRequest -UseBasicParsing https://raw.githubusercontent.com/vyta/windows-containers-workshop/kubernetes/labs/lab-04/monitoring/PrepWindowsNodesForAzMon.ps1 -OutFile PrepWindowsNodesForAzMon.ps1;
    powershell PrepWindowsNodesForAzMon.ps1
    ```
  1. Then to install the Windows Agents on our Windows Nodes:

      1. In the Azure portal, click All services found in the upper left-hand corner. In the list of resources, type Log Analytics. As you begin typing, the list filters based on your input. Select Log Analytics.
      1. In your list of Log Analytics workspaces, select DefaultLAWorkspace created earlier.
      1. On the left-hand menu, under Workspace Data Sources, click Virtual machines.
      1. In the list of Virtual machines, select a virtual machine you want to install the agent on. Notice that the Log Analytics connection status for the VM indicates that it is Not connected.
      1. In the details for your virtual machine, select Connect. The agent is automatically installed and configured for your Log Analytics workspace. This process takes a few minutes, during which time the Status is Connecting.
      1. After you install and connect the agent, the Log Analytics connection status will be updated with This workspace.

## Part 3 - Working with Linux and Windows Workloads

## Part 4 - Deploy an Ingress controller

https://github.com/Azure/aks-engine/blob/077f396b69fd5674fc05e130fe8780045f136d9d/docs/howto/mixed-cluster-ingress.md