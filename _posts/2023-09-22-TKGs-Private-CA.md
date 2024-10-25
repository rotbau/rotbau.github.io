---
layout: archive
title: Trusting a Private CA on TKGs clusterclass clusters
categories: [kubernetes,tkgs,tanzu]
---

Many organizations will use their private CA to sign certificates for various applications including their image registry.  And unless you tell docker or containerd or kapp to trust the certificate bad things happen.  Anyone who has ever run up against the dreaded "X. 509 Certificate Signed by Unknown Authority" when trying to do a docker pull or seeing your pod images stuck in imagePullBackOff will know exactly what I'm talking about.

Tanzu Kubernetes Grid (vSphere with Tanzu, TKGs) on vSphere 8 now uses class based clusters versus the old Tanzu Kubernetes Cluster. The method to add an additionalCA trust is not well documented for cluster class TKG clusters.  The basic premise is you create a secret in the vSphere namespace the cluster will be created in, and then add that secret to the cluster manifest and create the cluster.

This is example is for adding a CA for a registry but could be any CA you need to trust.  If you have multiple, concatenate the certificates in the secret.

## Step One - Create Secret in vSphere Namespace

1. Secret created needs to named <clustername>-user-trusted-ca-secret in the vSphere namespace where the cluster will be created
2. Obtain the ca.crt that Registry was signed by
3. Double base64 encode the certficate (yeah you read that right). If the CA has intermediate and root have them all in the same file and then double base 64 that.
```
base64 -w 0 ca.crt | base64 -w 0
```
4. Repeat for any other CAs you need to add (example below adds harbor-ca:, ca-1:, ca-2)
5. Create Manifest for trusted-ca-secret and add the name of the certificate (harbor-ca) and the double base64 encoded ca.crt.  The name can be anything but needs to match what you put in the cluster manifest trust section.
```
cat > <clustername>-user-trusted-ca-secret.yaml <<-EOF
apiVersion: v1
kind: Secret
metadata:
  name: <clustername>-user-trusted-ca-secret
  namespace: <vsphere-namespace>
data:
  harbor-ca: TFMwdExTMUNSVWRako0Y0dONlJWWk5RazFIUVRGVlJRcERaM2............
  ca-1: TCARMdkeiADMMEKdkeSlCRFJWSlVTVVpKUTBGVVJTMHDNMESSEdceD............
  ca-2: TDFWdcaeEGEUNEDvdFEFdddVGlCRFJWSlVTVVpKUTBGVVJDFDEGEdc............
type: Opaque
EOF
```
6. Apply trusted ca manifest in vsphere namespace context
```
kubectl apply -f <clustername>-user-trusted-ca-secret.yaml
```

## Update TKG Cluster Manifest variables section

1. Modify the variables section to add the trust section
```
spec:
  variables:
    - name: vmClass
      value: tmc-vm
    - name: storageClass
      value: vsan-default-storage-policy
    - name: defaultStorageClass
      value: vsan-default-storage-policy
    - name: trust
      value:
        additionalTrustedCAs:  #array so you can add many name/value pairs
        - name: harbor-ca   # name needs to match the name in the secret data section
        - name: ca-1
        - name: ca-2
```
2. Create the cluster
3. Optional: Verify the cluster has the certificate
```
vsphere namesspace context
kubectl get cluster <clustername> -oyaml |grep -i -A 5 additionaltrustedcas
verify harbor-ca is present
```

## Update existing TKG Cluster to add or replace Trusted CA

If you need to replace an existing TKG cluster with a new Trusted CA, or you need to add addtional CAs to the cluster you can follow this process.

1. Double base64 encode the new or addtional CA certificate you wish to trust 
```
base64 -w 0 ca.crt | base64 -w 0
```
2. Edit the <clustername>-user-trusted-ca-secret yaml file you initially used to create the secret and either replace the existing CA value or add a line for the new CA as shown below
```
apiVersion: v1
kind: Secret
metadata:
  name: <clustername>-user-trusted-ca-secret
  namespace: <vsphere-namespace>
data:
  harbor-ca: TFMwdExTMUNSVWRako0Y0dONlJWWk5RazFIUVRGVlJRcERaM2............
  new-ca: TCARMdkeiADMMEKdkeSlCRFJWSlVTVVpKUTBGVVJTMHDNMESSEdceD............
type: Opaque
```
3. Apply the updated secret yaml in the vSphere namespace that houses cluster objects
```
kubectl config use-context <vsphere namespace for cluster>
kubectl apply -f <clustername>-user-trusted-ca-secret yaml
secret/<clustername>-user-trusted-ca-secret configured
```
4. Edit the TKG cluster object to add the new CA secret name
```
kubectl config use-context <vsphere namespace for cluster>
kubectl get clusters
kubectl edit cluster <clustername>
```
5. In the cluster object (or manifest) add the new trusted CA secret name under the spec.topology.variables.name.trust section
```
...
spec:
  variables:
    - name: trust
      value:
        additionalTrustedCAs:
        - name: harbor-ca
        - name: new-ca
```
6. Either apply the manifest if you edited that or save the edited cluster object.  If this is succesful you should see something similar to 
```
cluster.cluster.x-k8s.io/<clustername> edited
```
7. Once the dchanges are applied you will notice the TKG nodes in the clusters will be replaced using a rolling upgrade.  Once all the nodes in the cluster have been replaced the operation is complete
8. You can validate the new secret is present
```
kubectl get cluster <clustername> -oyaml |grep -i -A 5 additionaltrustedcas

        additionalTrustedCAs:
        - name: harbor-ca
        - name: new-ca
```

**NOTE:  If you are upgrading an existing Trusted CA (for example we want to replace the harbor-ca) you may want to create the Secret with a new name (harbor-new-ca) so that the cluster recognizes the value has changed and will trigger the rolling upgrade**

**NOTE:  If you delete the cluster, the {cluster}-tmcsm-user-trusted-ca-secret will also be deleted.  So you need to recreate the secret before you recreate the cluster**

You should now be able to successfully pull images from your private registry to TKGs workload cluster.  Note today there is NO easy way to allow the supervisor cluster to also trust a private certificate.  It can be done manually but SSH'ing to SC nodes and putting the certificate there but this is not supported.

**Disclaimer:** All posts, contents and examples are for educational purposes only and does not constitute professional advice. No warranty and user excepts All information, contents, opinions are my own and do not reflect the opinions of my employer. Most likely you shouldn’t listen to what I’m saying and should close this browser window immediately