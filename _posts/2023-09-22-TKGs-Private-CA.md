---
layout: post
title: Trusting a Private CA on TKGs clusterclass clusters
description:
tags:
- tanzu
- kubernetes
- tkgs
---

Many organizations will use their private CA to sign certificates for various applications including their image registry.  And unless you tell docker or containerd or kapp to trust the certificate bad things happen.  Anyone who has ever run up against the dreaded "X. 509 Certificate Signed by Unknown Authority" when trying to do a docker pull or seeing your pod images stuck in imagePullBackOff we know exactly what I'm talking about.

Tanzu Kubernetes Grid (vSphere with Tanzu, TKGs) on vSphere 8 now uses class based clusters versus the old Tanzu Kubernetes Cluster. The method to add an additionalCA trust is not well documented for cluster class TKG clusters.  The basic premise is you create a secret in the vSphere namespace the cluster will be created in, and then add that secret to the cluster manifest and create the cluster.

This is example is for adding a CA for a registry but could be any CA you need to trust.  If you have multiple, concatenate the certificates in the secret.

## Step One - Create Secret in vSphere Namespace

1. Secret created needs to named {clustername-user-trusted-ca-secret} in the vSphere namespace where the cluster will be created
2. Obtain the ca.crt that Registry was signed by
3. Create Manifest for trusted-ca-secret
```
cat > clustername-user-trusted-ca-secret.yaml <<-EOF
apiVersion: v1
kind: Secret
metadata:
  name: {cluster}-tmcsm-user-trusted-ca-secret
  namespace: {vsphere-namespace}
data:
  harbor-ca: TFMwdExTMUNSVWRKVGlCRFJWSlVTVVpKUTBGVVJTMHRMUzB0Q2sxSlNVWnhSRU5EUVRWRFowRjNTVUpCWjBsQ1FVUkJUa0puYTNGb2EybEhPWGN3UWtGUk1FWkJSRUpzVFZGemQwTlJXVVJXVVZGSFJYZEtWbFY2UlZNS1RVSkJSMEV4VlVWRFFYZEtWRmRzZFdKdFZucGlNMUpvVFZKUmQwVm5XVVJXVVZGSVJFRjBUbUZYTlhWYVYwWjNZako0Y0dONlJWWk5RazFIUVRGVlJRcERaM2............
type: Opaque
EOF
```
4. Apply trusted ca manifest in vsphere namespace context
```
kubectl apply -f clustername-user-trusted-ca-secret.yaml
```

## Update TKG Cluster Manifest variables section

1. Modify the variables section to add the trust section
```
    variables:
      - name: vmClass
        value: tmc-vm
      - name: storageClass
        value: vsan-default-storage-policy
      - name: defaultStorageClass
        value: vsan-default-storage-policy
      - name: trust
        value:
          additionalTrustedCAs:
          - name: harbor-ca   ### name needs to match the name in the secret data section
```
2. Create the cluster
3. Optional: Verify the cluster has the certificate
```
vsphere namesspace context
kubectl get cluster {clustername} -oyaml |grep -i -A 5 additionaltrustedcas
verify harbor-ca is present
```
