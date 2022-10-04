- on Linux

~~~
## Copy your Root CA to the sources path
cp root-ca.pem /etc/pki/ca-trust/source/anchors/

## Update the Root CA Trust Bundles
update-ca-trust
~~~

1. Bake it in Container Image

2. Configmaps and Volume Mounts

~~~
## Create a ConfigMap YAML file for/from the PEM-encoded Root CA bundle
oc create configmap tls-ca-bundle-pem --from-file=/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem -o yaml --dry-run=client > cfgmap-tls-ca-bundle-pem.yml

## Create a ConfigMap YAML file for/from the Java Keystore
oc create configmap jks-ca-certs --from-file=/etc/pki/ca-trust/extracted/java/cacerts -o yaml --dry-run=client > cfgmap-jks-ca-certs.yml
~~~

3. Admission Controllers and MutatingWebhooks - PodPreset Operator

- An example of a ValidationWebhook would be to ensure Namespaces don’t have certain phrases 
- MutatingWebhooks mutate objects so a MutatingWebhook could certainly automatically apply Volumes and VolumeMounts to Pods as they’re applied to the cluster! 

~~~
## Clone down the repo
git clone https://github.com/redhat-cop/podpreset-webhook

## Enter the directory
cd podpreset-webhook

## Deploy the Operator to the cluster
make deploy IMG=quay.io/redhat-cop/podpreset-webhook:latest

## Check to make sure the Operator is applied and the PodPreset CRD is available
oc get crd/podpresets.redhatcop.redhat.io
~~~


Cluster Wide Solution:

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.10/html/networking/configuring-a-custom-pki

- creating/modifying a ConfigMap called `user-ca-bundle` in the `openshift-config` Namespace with a `.data['ca-bundle.crt']` entry, similar to below if you were adding 3 new Root CAs:

~~~
apiVersion: v1
data:
  ca-bundle.crt: |
    -----BEGIN CERTIFICATE-----
    MIIGqzCCBJOgAwIBAgIUKMZCYZxHomZOUFLz8j0/ItBY/3cwDQYJKoZIhvcNAQEL
    BQAwgdwxKzApBgkqhkiG9w0BCQEWHGNlcnRtYXN0ZXJAcG9seWdsb3QudmVudHVy
    ...
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MIIGqzCCBJOgAwIBAgIUKMZCYZxHomZOUFLz8j0/ItBY/3cwDQYJKoZIhvcNAQEL
    BQAwgdwxKzApBgkqhkiG9w0BCQEWHGNlcnRtYXN0ZXJAcG9seWdsb3QudmVudHVy
    ...
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MIIGqzCCBJOgAwIBAgIUKMZCYZxHomZOUFLz8j0/ItBY/3cwDQYJKoZIhvcNAQEL
    BQAwgdwxKzApBgkqhkiG9w0BCQEWHGNlcnRtYXN0ZXJAcG9seWdsb3QudmVudHVy
    -----END CERTIFICATE-----
kind: ConfigMap
metadata:
  name: user-ca-bundle
  namespace: openshift-config
~~~

Once that ConfigMap is applied, you would modify the cluster’s Proxy configuration and provide the user-ca-bundle ConfigMap name:

~~~
apiVersion: config.openshift.io/v1
kind: Proxy
metadata:
  name: cluster
spec:
  trustedCA:
    name: user-ca-bundle
~~~

- Now OCP cluster can automatically sync the build Root CA bundle in PEM format to an empty ConfigMap that has been given an annotation as such:

~~~
kind: ConfigMap
apiVersion: v1
metadata:
  name: trusted-ca
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true'
data: {}
~~~

Even though this ConfigMap is empty, once applied you’ll find the `.data['ca-bundle.crt']` data automatically added and it will be kept synced as Root CAs are updated centrally in the `user-ca-bundle` ConfigMap in the `openshift-config` Namespace!

Keep in mind - this is just for automatic syncing of Root CA Bundles in PEM format.

What about for automatic syncing of a Java Keystore embedded in a binaryData key in a ConfigMap?

- Reflector - sync ConfigMaps and Secrets across Namespaces:

~~~
## Add emberstack repo 
helm repo add emberstack https://emberstack.github.io/helm-charts

## Update Helm repos
helm repo update

## Create a reflector project
oc new-project reflector

## Install the Helm chart
helm upgrade --install reflector emberstack/reflector --namespace reflector

## Add SCC to SA RBAC
oc adm policy add-scc-to-user privileged -z default -n reflector
oc adm policy add-scc-to-user privileged -z reflector -n reflector
~~~

- Create a Namespace that holds those ConfigMaps centrally:

~~~
## Create a new Project
oc new-project pki-resources

## Create a ConfigMap YAML file for/from the PEM-encoded Root CA bundle
oc create configmap tls-ca-bundle-pem --from-file=/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem

## Create a ConfigMap YAML file for/from the Java Keystore
oc create configmap jks-ca-certs --from-file=/etc/pki/ca-trust/extracted/java/cacerts
~~~

~~~
## Add annotations for enabling reflection
oc annotate configmap tls-ca-bundle-pem reflector.v1.k8s.emberstack.com/reflection-allowed="true"
oc annotate configmap jks-ca-certs reflector.v1.k8s.emberstack.com/reflection-allowed="true"

## Add annotations for allowed Namespaces, allowed Namespaces being all of them
oc annotate configmap tls-ca-bundle-pem reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces=".*"
oc annotate configmap jks-ca-certs reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces=".*"
  
## Add annotations for enabling auto reflection
oc annotate configmap tls-ca-bundle-pem reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true"
oc annotate configmap jks-ca-certs reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true"
~~~

==================================================================

1. Add your Root CA in PEM format to the cluster which will automatically be baked and distributed to the nodes, consumed by cluster operators, and made available to be synced to an annotated ConfigMap.

2. Create a Namespace for the central store of PKI assets and create that annotated ConfigMap:

~~~
## Create a new project for PKI assets
oc new-project pki-resources

## Create a ConfigMap that will have the Root CA Bundle in PEM format synced to it
cat <<EOF | oc apply -f -
kind: ConfigMap
apiVersion: v1
metadata:
  name: trusted-ca
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true'
data: {}
EOF
~~~

- Add the Java Keystore ConfigMap - this will need to be manually managed since the cluster doesn’t sync a JKS like it does a PEM bundle:

~~~
## Assuming your Root CAs are part of the system root trust bundle...
oc create configmap jks-ca-certs --from-file=/etc/pki/ca-trust/extracted/java/cacerts
~~~

3. Deploy Reflector and the PodPreset Operator and label the two ConfigMaps for reflection across all Namespaces:

~~~
## Add annotations for enabling reflection
oc annotate configmap trusted-ca reflector.v1.k8s.emberstack.com/reflection-allowed="true"
oc annotate configmap jks-ca-certs reflector.v1.k8s.emberstack.com/reflection-allowed="true"

## Add annotations for allowed Namespaces, allowed Namespaces being all of them
oc annotate configmap trusted-ca reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces=".*"
oc annotate configmap jks-ca-certs reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces=".*"
  
## Add annotations for enabling auto reflection
oc annotate configmap trusted-ca reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true"
oc annotate configmap jks-ca-certs reflector.v1.k8s.emberstack.com/reflection-auto-enabled="true"
~~~

- Now PEM and JKS ConfigMaps available in every Namespace

- To use the ConfigMaps as set above, the PodPreset would look like this:

~~~
apiVersion: redhatcop.redhat.io/v1alpha1
kind: PodPreset
metadata:
  name: pki-volumes
spec:
  selector:
    matchLabels:
      inject-pki: "yes"
  volumeMounts:
    - mountPath: /etc/pki/ca-trust/extracted/pem
      name: trusted-ca
      readOnly: true
    - mountPath: /etc/pki/ca-trust/extracted/java
      name: jks-ca-certs
      readOnly: true
  volumes:
    - configMap:
        items:
          - key: ca-bundle.crt
            path: tls-ca-bundle.pem
        name: trusted-ca
      name: trusted-ca
    - configMap:
        items:
          - key: cacerts
            path: cacerts
        name: jks-ca-certs
      name: jks-ca-certs
~~~

- Label the workloads you need with that `inject-pki: "yes"` selector:

~~~
kind: Deployment
apiVersion: apps/v1
metadata:
  name: pki-toolbox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pki-toolbox
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: pki-toolbox
        inject-pki: "yes"
    spec:
      containers:
        - name: pki-toolbox
          image: 'quay.io/rh_ee_svasilic/pki-tools:latest'
          command:
            - /bin/bash
            - '-c'
            - '--'
          args:
            - while true; do sleep 30; done;
~~~

Steps:

- Root CA PEM Bundled by the Cluster, automatically synced to a ConfigMap in a central Namespace
- A Java Keystore in a central Namespace
- Automatic syncing of those ConfigMaps to other namespaces with Reflector
- Automatic injection of those namespaced ConfigMaps with a PodPreset
- Label workloads 


