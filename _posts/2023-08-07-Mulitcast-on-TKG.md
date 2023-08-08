---
layout: post
title: Enabling Multicast on Tanzu TKGs with Antrea
---

I recently worked with a customer on testing multicast patterns (Pod -> Pod and Pod -> VM) on Tanzu TKGs with antrea.  Antrea is an OSS Kubernetes CNI developed by VMware and is currently a CNCF sandbox project.  The use case was around Tanzu Kuberntes Grid Supervisor (TKGs) which is Kubernetes integrated into vSphere.

Antrea CNI multicast support is Alpha, so the feature gate needs to be enabled to use it.  This is normally done via an antrea-config configmap object.  However TKGs supervisor controls these objects and setting the antrea-config cm directly will result in the changes being reverted by the supervisor.  As of vSphere 8.0 update 1, you can create an AntreaConfig object that allows you to modify the configuration of Antrea per TKG workload cluster.

I'm not going to dig into vSphere with Tanzu components, setup or configuration.  Please reference [VMware's official documentation](https://docs.vmware.com/en/VMware-vSphere/index.html){:target="_blank"} to learn all about vSphere with Tanzu.  Also note that Tanzu Kuberentes Grid with Standalone Management Cluster support directly editing the antrea-config configmap to change the Antrea configuration.  Refer to [Antrea upstream documents](https://antrea.io/docs/v1.12.1/docs/feature-gates/){:target="_blank"} for that.

## Test Setup
- vCenter 8.0u1c
- vSphere ESXi 8.0u1a
- Tanzu Kuberentes Grid (TKG) Supervisor enabled
- TKG v1.25 Ubuntu based Kuberentes Cluster

**Note about versions:**  The antreaconfig object that is used to customize antrea feature gates was introduced in vSphere 8.0u1+.  This allows you to manage the configmap that antrea uses for configuration per TKG workload cluster.  Without getting too deep, you cannot just manually edit the antrea-config configmap on the TKG workload clusters becaause the antrea controller running on the supervisor cluster will overwrite those changes.  This is a common pattern on TKG supervisor managed clusters.  In vSphere 8.0u1 attempting to add the Multicast: true to the antreaconfig feature gate generated an error that Mulitcast featuregate was unknown.  Because of this I put vSphere 8.0u1c as the version.  It may work on older u1 versions.

## Create the Antra Config in the vSphere Namespace where TKG cluster will be created

Create a vSphere namespace.  I'm using the vsphere namespace `mcast` for this example
Change to mcast context `kubectl config use-context mcast`
Create a AntreaConfig object in the format of clustername-antrea-package in the vSphere namespace where cluster will be created.
  - AntreaConfig required object name format `clustername-antrea-package`
  - AntreaConfig Example Yaml File with `Multicast: true` and `NetworkPolicyStats: true`

```
cat <<EOF > mccluster-antrea-config.yaml
apiVersion: cni.tanzu.vmware.com/v1alpha1
kind: AntreaConfig
metadata:
  name: mccluster-antrea-package
  namespace: mcast
spec:
  antrea:
    config:
      featureGates:
        AntreaProxy: true
        EndpointSlice: false
        AntreaPolicy: true
        FlowExporter: true
        Egress: true
        NodePortLocal: true
        AntreaTraceflow: true
        NetworkPolicyStats: true
        Multicast: true
EOF
```
Create AntreaConfig `kubectl apply -f mccluster-antrea-config.yaml`
Validate AntreaConfig was created in vSphere namespace mccast `kubectl get antreaconfig`

## Create TKG Workload Cluster

Next steps is to create the TKG Workload cluster in the mcast vSphere Namespace with the same name used as part of the antrea-package name.  In our example this is mccluster (mccluster-antrea-package).  The TKG v1.25 includes Antrea 1.9.0 which supports multicast with encap so this is why we want to use v1.25 TKR.  

It is important to use the Ubuntu OS for the cluster instead of Photon.  When testing with Photon the antrea-agent pods went into crash loopback with an error 'failed to create multicast socket' so it appears there is currently an issue with Photon.

**Example Ubuntu v1.25 TKG Workload Cluster Manifest**

```
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mccluster
  namespace: mcast
spec:
  clusterNetwork:
    services:
      cidrBlocks: ["198.51.100.0/12"]
    pods:
      cidrBlocks: ["192.0.2.0/16"]
    serviceDomain: "cluster.local"
  topology:
    class: tanzukubernetescluster
    version: v1.25.7---vmware.3-fips.1-tkg.1
    controlPlane:
      replicas: 1
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu
    workers:
      machineDeployments:
        - class: node-pool
          name: node-pool-1
          replicas: 2
          metadata:
            annotations:
              run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu
    variables:
      - name: vmClass
        value: best-effort-xsmall
      - name: storageClass
        value: vsan-default-storage-policy
      - name: defaultStorageClass
        value: vsan-default-storage-policy
      - name: clusterEncryptionConfigYaml
        value: |
          apiVersion: apiserver.config.k8s.io/v1
          kind: EncryptionConfiguration
          resources:
            - resources:
                - secrets
              providers:
                - aescbc:
                    keys:
                      - name: key1
                        secret: QiMgakjdfajoifa99fa0dxxxx.dkaafad
                - identity: {}
```

- Create Cluster `kubectl apply -f mccluster-classy.yaml`
- Authenticate to workload cluster using appropriate `kubectl vsphere login` commands
- Change to workload cluster context `kubectl config use-context mccluster`
- Verify antrea-agent and antrea-controller pods are running
```
kubectl get po -n kube-system |grep antrea
antrea-agent-7xg5t                                        2/2     Running   0          46h
antrea-agent-f8m2d                                        2/2     Running   0          46h
antrea-agent-xqxk5                                        2/2     Running   0          46h
antrea-controller-689b7cdfc5-4p6r9                        1/1     Running   0          46h
```
- Describe the antrea-configmap and verify Multicast and NetworkPolicyStats feature gates are set to true
```
kubectl describe cm antrea-config -n kube-system

antrea-controller.conf:
----
featureGates:
  Traceflow: true
  AntreaPolicy: true
  NetworkPolicyStats: true
  Multicast: true

antrea-agent.conf:
----
featureGates:
  AntreaProxy: true
  EndpointSlice: false
  TopologyAwareHints: false
  Traceflow: true
  NodePortLocal: true
  AntreaPolicy: true
  FlowExporter: true
  NetworkPolicyStats: true
  Egress: true
  AntreaIPAM: false
  Multicast: true

```
- Exec into the antrea-agent and antrea-controller pods directly and use antctl to check status
```
kubectl exec -ti antrea-agent-xqxk5 -n kube-system -- sh

# antctl get featuregates
Antrea Agent Feature Gates
FEATUREGATE              STATUS         VERSION
AntreaProxy              Enabled        BETA
NodePortLocal            Enabled        BETA
TopologyAwareHints       Disabled       ALPHA
AntreaIPAM               Disabled       ALPHA
NodeIPAM                 Disabled       ALPHA
Multicluster             Disabled       ALPHA
ExternalNode             Disabled       ALPHA
AntreaPolicy             Enabled        BETA
Traceflow                Enabled        BETA
TrafficControl           Disabled       ALPHA
Egress                   Enabled        BETA
EndpointSlice            Disabled       ALPHA
FlowExporter             Enabled        ALPHA
NetworkPolicyStats       Enabled        BETA
Multicast                Enabled        ALPHA

 ```
 ## Create Containers to test Mutlicast

 I'm using simple iperf to test Mutlicast sending / receiver.  I put together a couple simple Dockefiles to acheive this

 ### Sender Dockerfile

 ```
 mkdir sender
 cd sender
 cat <<EOF > Dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install -y tcpdump
RUN apt-get install -y iputils-ping
RUN apt-get install -y iputils-tracepath
RUN apt-get install -y iproute2
RUN apt-get install -y iperf
ENTRYPOINT ["/usr/bin/iperf", "-s", "-u", "-B", "239.255.12.43", "-i", "1"]
EOF
```

### Receiver Dockerfile

```
mkdir receiver
cd reciever
cat <<EOF > Dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install -y tcpdump
RUN apt-get install -y iputils-ping
RUN apt-get install -y iputils-tracepath
RUN apt-get install -y iproute2
RUN apt-get install -y screen
RUN apt-get install -y iperf
ENTRYPOINT ["/usr/bin/iperf", "-c", "239.255.12.43", "-u", "-t", "86400"]
EOF
```
- Build the docker images with approriate tags to your image registry and push them
```
docker build . -t registry/project/mcreceiver:v1
```

## Testing Pod-to-Pod Multicast - Same Cluster

### Create Pod Manifests

- Sender Pod
```
cat <<EOF > mcsender.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mcsender
  name: mcsender
  namespace: default
spec:
  containers:
  - image: {registry-fqdn}/mcsender:v1
    imagePullPolicy: IfNotPresent
    name: mcsender
EOF
```

- Receiver Pod
```
cat <<EOF > mcreceiver.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mcreceiver
  name: mcreceiver
  namespace: default
spec:
  containers:
  - image: {registry-fqdn}/mcreceiver:v1
    imagePullPolicy: IfNotPresent
    name: mcreceiver
EOF
```

### Deploy Pods
```
kubectl apply -f mcsender.yaml
kubectl apply -f mcreceiver.yaml
```

### Verify Multicast Traffic

The mcsender should already be sending traffic since the entrypoint of the container is running the iperf command.  To validate mutlticast traffic is working you can exec to the mcreceiver pod and manually run the iperf receiver commands to subscribe to the stream being sent by the mcsender.

Start Sender manually
- Exec to mcsender pod 
```
kubectl exec -ti mcsender -- /bin/bash
```
- Run the iperf server command to send multicast stream
```
iperf -s -u -B 239.255.12.43 -i 1
```

Start Receiver manually
- Exec to mcreceiver pod
```
kubectl exec -ti mcreceiver -- /bin/bash
```
- Run iperf client command to subscrib to multicast stream
```
iperf  -c 239.255.12.43 -u -t 86400
```
- If mutlicast is working you should see something like this
[mcreceiver pod](/images/mcreceiver-pod.png)

Check multicastgroups from K8s cli
```
kubectl get multicastgroups
GROUP           PODS
239.255.12.43   default/mcreceiver
```
- Exec into antrea-agent pods and view mulicast groups.  Note: you will only see information on the pods running on the same K8s node as the antrea-agent

**antrea-agent podmulticaststats for mcreceiver**
```
kubectl exec -ti antrea-agent-7xg5t -n kube-system -- sh

antctl get podmulticaststats

NAMESPACE            NAME                                  INBOUND OUTBOUND
default              mcreceiver                            827309  0
kube-system          metrics-server-588ff6d9b4-tlwpm       0       0
secretgen-controller secretgen-controller-75d88bc999-m7zkh 0       0
```

**atrea-agent podmulticaststats for mcsender**
```
kubectl exec -ti antrea-agent-xqxk5 -n kube-system -- sh

antctl get podmulticaststats

NAMESPACE NAME     INBOUND OUTBOUND
default   mcsender 0       450370
```

## Testing Pod-to-VM Multicast

The Pod sender in the CNI should support multicast traffic from POD -> VM but in my case this was not working.  As I troubleshoot this with Antrea there is a workaround where you can use hostNetwork on the mcsender pod.  This puts the Pod on the host network and Pod will have an ip address same as the kubernetes node.  While this is not ideal from a security standpoint (requires priviledged escalation) it does work.  I will update this post once I sort out why it doesn't work with hostNetwork

**mcsender pod with hostNetwork**

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mcsender
  name: mcsender
  namespace: default
spec:
  containers:
  - image: mcsender:v1
    imagePullPolicy: IfNotPresent
    name: mcsender
  dnsPolicy: ClusterFirstWithHostNet #ClusterFirst or ClusterFirstWithHostNet
  hostNetwork: true
```

Disclaimer: All posts, contents and examples are for educational purposes only and does not constitute professional advice. No warranty and user excepts All information, contents, opinions are my own and do not reflect the opinions of my employer. Most likely you shouldn’t listen to what I’m saying and should close this browser window immediately

