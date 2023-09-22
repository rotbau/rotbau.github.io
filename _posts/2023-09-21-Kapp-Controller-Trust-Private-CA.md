---
layout: post
title: Getting Kapp to Trust Private Repository on TKGs
description:
tags:
- kubernetes
- tanzu
- kapp
- tkgs
---

Kapp controller is Package management system for Kubernetes and part of the open-source Carvel Suite.  Essentially it allows you to life cycle manage applications on Kubernetes in the form of packages.  I won't get into the entire philosophy of how it works but head over to carvel.dev to learn more.  VMware has been using kapp to deploy a number of system packages for some time now.

Your application packages are hosted on a registry (Harbor or any container registry).  Quite a few organizations will host their container registries internally and have their certificates signed by their internal CA.  For those in the know that means making sure workstations, servers, Kubernetes and other items "trust" this CA.  For Kapp this means you need to give the ca.crt to the Kapp controller so it can connect to and sync package repositories.

There are several options for configuring kapp controller to trust package repositories hosted on private registries on vSphere 8 with Tanzu (TKGs).  

## Option One - Pre-creating kapp-controller-package object in vSphere Namespace

One of the cool things with Tanzu 2.0 on vSphere 8 and clusterclass clusters is you can pre-create objects in the vSphere namespace prior to creating the TKG workload cluster and these objects will be picked up by the tkr-controller and configured on the cluster during build.  One such object is the kapp-controller-package.  This allows you to set items like the proxy, noproxy, cacerts and skipTLS.

1. Authenticate to supervisor if you haven't and change context to the vsphere namespace where you plan to create the TKG workload cluster

2. Create kappcontrollerconfig manifest (example below for a cluster called tkg-cluster1 but suggestion you take a template from and existing cluster and add only the config.certs section as shown)
```
export clustername="tkg-cluster1"

cat > $clustername-kapp-controller-config.yaml <<-EOF
apiVersion: run.tanzu.vmware.com/v1alpha3
kind: KappControllerConfig
metadata:
  name: $clustername-kapp-controller-package
  namespace: tmc
spec:
  kappController:
    config:
      caCerts: |
        -----BEGIN CERTIFICATE-----
        MIIFqDCCA5CgAwIBAgIBADANBgkqhkiG9w0BAQ0FADBlMQswCQYDVQQGEwJVUzES
        MBAGA1UECAwJTWlubmVzb3RhMRQwEgYDVQQHDAtNaW5uZWFwb2xpczEVMBMGA1UE
        ...........
        -----END CERTIFICATE-----
    createNamespace: false
    deployment:
      apiPort: 10100
      concurrency: 4
      hostNetwork: true
      metricsBindAddress: "0"
      priorityClassName: system-cluster-critical
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        key: node.kubernetes.io/not-ready
      - effect: NoSchedule
        key: node.cloudprovider.kubernetes.io/uninitialized
        value: "true"
    globalNamespace: tkg-system
  namespace: tkg-system
EOF
```
3. Edit the kapp-controller-config.yaml and under spec.kappController.config.caCerts replace the value with the ca.crt your registry was signed with.  Note if you need to get the CA crt you can run something like the following example for harbor.  Note api may be different for other registries
```
wget -O harbor-ca.crt https://harbor.domain.com/api/v2.0/systeminfo/getcert --no-check-certificate
``` 
4. The name of the KappControllerConfig metadata.name MUST be in the format of {clustername}-kapp-controller-package
5. If you want to investigate other fields you can add under the spec.kappController.config run the following command from vsphere namespace context
```
kubectl explain kappcontrollerconfig.spec.kappController.config
```
5. Create the KappControllerConfig in the vSphere namespace where you will create the TKGs workload cluster
```
kubectl apply -f tkg-cluster1-kapp-controller-package.yaml

kubectl get kappcontrollerconfig

NAME                                NAMESPACE    GLOBALNAMESPACE   SECRETNAME
tkg-cluster1-kapp-controller-package   tkg-system   tkg-system
```
6. Create your TKG Cluster.  You of course will need your TKG cluster to also trust the CA for your registry if you intend to deploy containers from it.  This is covered in the vSphere with Tanzu documentation.
7. Once the cluster is created you can view the kapp configmap on the TKG workload cluster to verify your cacert is included
```
kubectl get cm -n tkg-system

NAME                               DATA   AGE
kapp-controller-config             5      5h19m
kube-root-ca.crt                   1      5h19m
tanzu-standard.pkgr                1      4h54m
tanzu-standard.pkgr-change-7hqbf   1      4h54m

kubectl describe cm kapp-controller-config -n tkg-system

Under Data section you will see which matches the ca.crt from above
caCerts:
----
-----BEGIN CERTIFICATE-----
MIIFqDCCA5CgAwIBAgIBADANBgkqhkiG9w0BAQ0FADBlMQswCQYDVQQGEwJVUzES
MBAGA1UECAwJTWlubmVzb3RhMRQwEgYDVQQHDAtNaW5uZWFwb2xpczEVMBMGA1UE
..........
```
## Option Two - Existing Cluster edit kappcontrollerconfig

1. Edit the existing kappcontrollerconfig for the cluster from the vSphere namespace context
```
kubectl get kappcontrollerconfig
kubectl edit kappcontrollerconfig tkg-cluster1-kapp-controller-package
```
2. Add the config.caCerts under spec.kappController like in the eample above and add your ca.crt
3. :wq the edit to save change.  You should see that the tkg-cluster1-kapp-controller-package was edited
4. Change context to TKG workload cluster
5. View kapp-controller-cconfig configmap to make sure caCerts is listed
6. Delete the kapp-controller pod from tkg-system namespace
7. Verify new kapp-controller pod goes running 2/2

## Option Three - Existing Cluster create kapp-controller-config secret

I tested this option and it also worked.  It seemed to take a little time for the kapp to trust the ca and reconcile packages but ultimately did work.  Note this should work for kapp on any Kubernetes not just Tanzu

1. Change to TKG workload cluster context
2. Create kapp-controller-conifig secret
```
cat > kapp-controller-secret.yaml <<-EOF
apiVersion: v1
kind: Secret
metadata:
  # Name must be `kapp-controller-config` for kapp controller to pick it up
  name: kapp-controller-config
  # Namespace must match the namespace kapp-controller is deployed to
  namespace: tkg-system
stringData:
  # A cert chain of trusted ca certs. These will be added to the system-wide
  # cert pool of trusted ca's (optional)
  caCerts: |
    -----BEGIN CERTIFICATE-----
    MIIFqDCCA5CgAwIBAgIBADANBgkqhkiG9w0BAQ0FADBlMQswCQYDVQQGEwJVUzES
    MBAGA1UECAwJTWlubmVzb3RhMRQwEgYDVQQHDAtNaW5uZWFwb2xpczEVMBMGA1UE
    ...........
    -----END CERTIFICATE-----
  # The url/ip of a proxy for kapp controller to use when making network
  # requests (optional)
  httpProxy: ""
  # The url/ip of a tls capable proxy for kapp controller to use when
  # making network requests (optional)
  httpsProxy: ""
  # A comma delimited list of domain names which kapp controller should
  # bypass the proxy for when making requests (optional)
  noProxy: ""
  # A comma delimited list of domain names for which kapp controller, when
  # fetching images or imgpkgBundles, will skip TLS verification. (optional)
  dangerousSkipTLSVerify: ""
EOF
```
3. Edit kapp-controller-secret.yaml and replace the caCerts with your own ca.crt value
4. Optionally fill in any of the other config options
5. Save file and apply to TKG workload cluster
```
kubectl apply -f kapp-controller-secret.yaml

kubectl get secrets -n tkg-system
```

**Important Note:  If you delete the cluster the custom KappControllerConfig will also be deleted so you will need to recreate before recreating the cluster.  There will still be a KappControllerConfig in the namespace but it will be the default one**

**Disclaimer:** All posts, contents and examples are for educational purposes only and does not constitute professional advice. No warranty and user excepts All information, contents, opinions are my own and do not reflect the opinions of my employer. Most likely you shouldn’t listen to what I’m saying and should close this browser window immediately
