
# Table of Contents

1.  [SKS Cluster Deployment](#org5c9f8a2)
    1.  [Deploy the demo cluster](#orgfff2e82)
    2.  [Download kubeconfig](#org36a744e)
    3.  [Install Longhorn](#orgdbbe1e8)
        1.  [Update the StorageClass name](#org9afebc5)
2.  [Deploying Calico Enterprise](#org1f48b2e)
    1.  [Deploy the Calico operator](#orge904cfc)
    2.  [Deploy the Prometheus Operator](#orgdf358dd)
    3.  [Install the pull secrets](#orge932295)
        1.  [Calico pull secrets](#org417ec66)
        2.  [Prometheus pull secrets](#org8c394a1)
    4.  [Install Calico custom resources](#org106711f)
    5.  [Create SecurityGroup](#org48aa760)
    6.  [Create nodepool](#orgec8cb0f)
    7.  [Deploy the License](#org995cce2)
    8.  [Calico Operator Workaround](#org201abf8)
        1.  [The SKS Kubernetes API server](#org0beeccd)
        2.  [Bypass network policy](#org1588083)
3.  [Accessissing the system](#orga5a3e96)
    1.  [Create a user](#orga21f8f0)
    2.  [Create an authentication token](#orgb1086f5)
    3.  [Port forward to Tigera Manager](#org2dab325)



<a id="org5c9f8a2"></a>

# SKS Cluster Deployment


<a id="orgfff2e82"></a>

## Deploy the demo cluster

    exo compute sks create ceosks-poc \
        --no-cni                      \
        --nodepool-size 0             \
        --zone at-vie-1

![img](deploy-sks.gif)


<a id="org36a744e"></a>

## Download kubeconfig

    rm -f "$KUBECONFIG"
    exo compute sks kubeconfig ceosks-poc admin \
        --zone at-vie-1                         \
        -g system:masters                       \
        -t $((86400 * 7)) > "$KUBECONFIG"
    chmod 0600 "$KUBECONFIG"

![img](download-kubeconfig.gif)


<a id="orgdbbe1e8"></a>

## Install Longhorn

    helm install my-longhorn longhorn \
         --version 1.5.1              \
         --repo https://charts.longhorn.io

![img](install-longhorn.gif)


<a id="org9afebc5"></a>

### Update the StorageClass name

Calico Enterprise wants to use the StorageClass
`tigera-elasticsearch`.

The file `storageclass-config.yaml` has the content:

    apiVersion: v1
    data:
      storageclass.yaml: |
        kind: StorageClass
        apiVersion: storage.k8s.io/v1
        metadata:
          name: tigera-elasticsearch
          annotations:
            storageclass.kubernetes.io/is-default-class: "true"
        provisioner: driver.longhorn.io
        allowVolumeExpansion: true
        reclaimPolicy: "Delete"
        volumeBindingMode: Immediate
        parameters:
          numberOfReplicas: "3"
          staleReplicaTimeout: "30"
          fromBackup: ""
          fsType: "ext4"
          dataLocality: "disabled"
    kind: ConfigMap
    metadata:
      annotations:
        meta.helm.sh/release-name: my-longhorn
        meta.helm.sh/release-namespace: default
      labels:
        app.kubernetes.io/instance: my-longhorn
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: longhorn
        app.kubernetes.io/version: v1.5.1
        helm.sh/chart: longhorn-1.5.1
      name: longhorn-storageclass
      namespace: default

    kubectl apply -f storageclass-config.yaml \
                  --server-side --force-conflicts

![img](update-storageclass.gif)


<a id="org1f48b2e"></a>

# Deploying Calico Enterprise


<a id="orge904cfc"></a>

## Deploy the Calico operator

    kubectl create -f https://downloads.tigera.io/ee/v3.17.2/manifests/tigera-operator.yaml

![img](deploy-calico-operator.gif)


<a id="orgdf358dd"></a>

## Deploy the Prometheus Operator

    kubectl create -f https://downloads.tigera.io/ee/v3.17.2/manifests/tigera-prometheus-operator.yaml

![img](deploy-prometheus-operator.gif)


<a id="orge932295"></a>

## Install the pull secrets

Calico Enterprise users need to be authenticated at the Calico
registry to download the container images.


<a id="org417ec66"></a>

### Calico pull secrets

    kubectl create secret generic tigera-pull-secret \
            --type=kubernetes.io/dockerconfigjson    \
    	-n tigera-operator                       \
    	--from-file=.dockerconfigjson=tigera-partners-origoss-auth.json

![img](install-calico-pull-secrets.gif)


<a id="org8c394a1"></a>

### Prometheus pull secrets

    kubectl create secret generic tigera-pull-secret \
            --type=kubernetes.io/dockerconfigjson    \
    	-n tigera-prometheus                     \
            --from-file=.dockerconfigjson=tigera-partners-origoss-auth.json
    
    kubectl patch deployment -n tigera-prometheus calico-prometheus-operator \
            -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name": "tigera-pull-secret"}]}}}}'

![img](install-prometheus-pull-secrets.gif)


<a id="org106711f"></a>

## Install Calico custom resources

    kubectl create -f https://downloads.tigera.io/ee/v3.17.2/manifests/custom-resources.yaml

![img](install-calico-custom-resources.gif)


<a id="org48aa760"></a>

## Create SecurityGroup

This security group opens the ports required by Calico Enterprise.

    exo compute security-group create ceosks-poc
    
    exo compute security-group rule add ceosks-poc \
                    --security-group ceosks-poc    \
    		--protocol tcp                 \
    		--port 179
    exo compute security-group rule add ceosks-poc \
                    --security-group ceosks-poc    \
    		--protocol udp                 \
    		--port 4789
    exo compute security-group rule add ceosks-poc \
                    --security-group ceosks-poc    \
    		--protocol tcp                 \
    		--port 5473
    exo compute security-group rule add ceosks-poc \
                    --security-group ceosks-poc    \
    		--protocol tcp                 \
    		--port 10250
    exo compute security-group rule add ceosks-poc \
                    --network 0.0.0.0              \
    		--protocol tcp                 \
    		--port 30000-32767
    exo compute security-group rule add ceosks-poc \
                    --network 0.0.0.0              \
    		--protocol udp                 \
    		--port 30000-32767
    exo compute security-group rule add ceosks-poc \
                    --security-group ceosks-poc    \
    		--protocol udp                 \
    		--port 51820-51821

![img](create-securitygroup.gif)


<a id="orgec8cb0f"></a>

## Create nodepool

    exo compute sks nodepool add \
        --zone at-vie-1 ceosks-poc ceosks-poc-worker \
        --size=2 \
        --instance-type c6f99499-7f59-4138-9427-a09db13af2bc \
        --security-group ceosks-poc

![img](create-nodepool.gif)


<a id="org995cce2"></a>

## Deploy the License

This is the Calico Enterprise license.

    kubectl create -f license.yml

![img](deploy-license.gif)


<a id="org201abf8"></a>

## Calico Operator Workaround

The Calico NetworkPolicies generated by the Calico Operator preventing
the components from reaching the SKS Kubernetes API server.


<a id="org0beeccd"></a>

### The SKS Kubernetes API server

    kubectl describe endpoints/kubernetes -n default

![img](sks-apiserver.gif)

The API server can be reached at `https://194.182.185.29:30876`. This
endpoint is not allowed by the default Calico network policies.


<a id="org1588083"></a>

### Bypass network policy

    apiVersion: projectcalico.org/v3
    kind: NetworkPolicy
    metadata:
      name: allow-tigera.allow-sks-apiserver
    spec:
      order: 0
      tier: allow-tigera
      types:
        - Egress
      egress:
        - action: Allow
          protocol: TCP
          destination:
            ports:
              - 30876

    kubectl apply -f bypass-networkpolicy.yaml -n calico-system
    kubectl apply -f bypass-networkpolicy.yaml -n tigera-eck-operator

![img](deploy-bypass-policies.gif)


<a id="orga5a3e96"></a>

# Accessissing the system


<a id="orga21f8f0"></a>

## Create a user

    kubectl create sa tigera-admin -n default
    kubectl create clusterrolebinding tigera-admin \
            --clusterrole tigera-network-admin     \
            --serviceaccount default:tigera-admin

![img](create-admin-user.gif)


<a id="orgb1086f5"></a>

## Create an authentication token

    kubectl create token tigera-admin -n default

![img](create-auth-token.gif)


<a id="org2dab325"></a>

## Port forward to Tigera Manager

    kubectl port-forward -n tigera-manager service/tigera-manager 9443:9443

Access the Tigera Manager dashboard at <https://localhost:9443>

![img](port-forward.gif)

