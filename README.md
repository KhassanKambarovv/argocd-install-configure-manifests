# ArgoCD-install-manifests

## What is this ArgoCD-vault-plugin?

Argo team introduced argocd-vault-plugin. This plugin is aimed at helping to solve the issue of secret management with GitOps and Argo CD. Using this plugin one can easily utilize Vault without having to rely on an operator or custom resource definition.
This plugin can be used not just for secrets but also for deployments, configMaps or any other Kubernetes resource. Within ArgoCD, there is a way to integrate custom plugins if you need something outside of the supported tools that are built-in. The plugin works by first retrieving values from Vault based on a path that can be specified as an Environment Variable or an Annotation inside of the YAML file.

## About Vault installation
I could start with the installation with Vault but unfortunately or fortunately to me it was installed in one of our Virtual Machine, so i will skip it, maybe in future i will write about this also maybe i will add some manifests for it.

## Getting started

The first thing we need to create argocd namespace and in that we will install the Argocd and integrate with Vault

## VAULT TOKEN AUTHENTICATION

For Vault Token Authentication, these are the required parameters:

```
VAULT_ADDR: Your HashiCorp Vault Address
VAULT_TOKEN: Your Vault token
AVP_TYPE: vault
AVP_AUTH_TYPE: token
```

and with this credential configurations we need to create secrete

```
apiVersion: v1
kind: Secret
metadata:
  name: vault-configuration
  namespace: argocd
data:
  VAULT_ADDR: aHR0cDovL3ZhdWx0LmRlZmF1bHQ6ODIwMA==
  VAULT_TOKEN: aHZzLkRQekVyQnFQaVQ2clRhSnA4WVUyZUFkVg==
  AVP_TYPE: dmF1bHQ=
  AVP_AUTH_TYPE: dG9rZW4=
type: Opaque
```

VAULT_ADDR is http://vault.default:8200 as we are running in the default namespace. Argocd can communicate between pods in the same cluster.
VAULT_TOKEN is the token we created earlier using the vault token create command.
AVP_TYPE is vault and AVP_AUTH_TYPE is token as we are using token-based authentication.

also you can find the manifest (**vault-config-secret.yaml**) with this document

**Configmap:**

There are two ways to integrate plugins using Installation via argocd-cm ConfigMap and other is Installation via a sidecar container. As the Installation via argocd-cm ConfigMap is deprecated in argocd version 2.6, We will choose the option based on sidecar and initContainer.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-plugin
  namespace: argocd
data:
  avp-helm.yaml: |
    ---
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: argocd-vault-plugin-helm
    spec:
      allowConcurrency: true
      discover:
        find:
          command:
            - sh
            - "-c"
            - "find . -name 'Chart.yaml' && find . -name 'values.yaml'"
      generate:
       command:
         - bash
         - "-c"
         - |
           helm template $ARGOCD_APP_NAME -n $ARGOCD_APP_NAMESPACE -f $ARGOCD_ENV_RELEASE_VALUES . |
           argocd-vault-plugin generate -s vault-configuration -
           

      lockRepo: false
  avp.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: argocd-vault-plugin
    spec:
      allowConcurrency: true
      discover:
        find:
          command:
            - sh
            - "-c"
            - "find . -name '*.yaml' | xargs -I {} grep \"<path\\|avp\\.kubernetes\\.io\" {} | grep ."
      generate:
        command:
          - argocd-vault-plugin
          - generate
          - "."
          - "-s"
          - "vault-configuration"
      lockRepo: false
---
```

This config map has two plugin configuration defined. One configuration defines the plugin is used if we are using helm templates and want to make use of argocd-vault-plugin for env variables. Another plugin configuration is for the plane k8 manifest files which can we stored in github itself. So these secret and configmap should be created inside the argocd namespace so that the argocd reposerver reaches it.

you can find the manifest (**vault-cmp-plugin.yaml**) with this document

next make some command for apply this manifests with crating namespace:
```
kubectl create ns argocd
kubectl apply -f secret.yaml -n argocd
kubectl apply -f cmp-plugin.yaml -n argocd

```

After this, To patch the argocd-repo-server to add an initContainer to download argocd-vault-plugin and define the sidecar, we install argoCD with with custom values.yaml.

```
repoServer:
  rbac:
    - verbs:
        - get
        - list
        - watch
      apiGroups:
        - ''
      resources:
        - secrets
        - configmaps
  initContainers:
    - name: download-tools
      image: registry.access.redhat.com/ubi8
      env:
        - name: AVP_VERSION
          value: 1.11.0
      command: [sh, -c]
      args:
        - >-
          curl -L https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v$(AVP_VERSION)/argocd-vault-plugin_$(AVP_VERSION)_linux_amd64 -o argocd-vault-plugin &&
          chmod +x argocd-vault-plugin &&
          mv argocd-vault-plugin /custom-tools/
      volumeMounts:
        - mountPath: /custom-tools
          name: custom-tools

  extraContainers:
    # argocd-vault-plugin with plain YAML
    - name: avp
      command: [/var/run/argocd/argocd-cmp-server]
      image: quay.io/argoproj/argocd:v2.4.0
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
      volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
        - mountPath: /tmp
          name: tmp

        # Register plugins into sidecar
        - mountPath: /home/argocd/cmp-server/config/plugin.yaml
          subPath: avp.yaml
          name: cmp-plugin

        # Important: Mount tools into $PATH
        - name: custom-tools
          subPath: argocd-vault-plugin
          mountPath: /usr/local/bin/argocd-vault-plugin
    
    - name: avp-helm
      command: [/var/run/argocd/argocd-cmp-server]
      image: quay.io/argoproj/argocd:v2.5.3
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
      volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
        - mountPath: /tmp
          name: tmp
        - mountPath: /home/argocd/cmp-server/config/plugin.yaml
          subPath: avp-helm.yaml
          name: cmp-plugin
        - name: custom-tools
          subPath: argocd-vault-plugin
          mountPath: /usr/local/bin/argocd-vault-plugin
      
  volumes:
    - configMap:
        name: cmp-plugin
      name: cmp-plugin
    - name: custom-tools
      emptyDir: {}
```
this manifest (**argo-vault-plugin.yaml**) with this document

Now we can install argocd with these argo-vault-plugin.yaml using:

```
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd -f argo-vault-plugin.yam
```

It should be installed successfully!

Next steps is to bound host, for that we can use the next manifest (**argocd-ingress.yaml**)

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
spec:
  ingressClassName: traefik
  rules:
  - host: argo-host
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

## Connecting other clusters to argo (MultiCluster)

For that we need to create serviceAccount in cluster which we want to add to argo:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-manager
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: argocd-manager
  namespace: kube-system 

```
Also you can fin next to this docs (**argo-cluster-role-binding.yaml**)

And we need to take token and ca cert for that:

1. Get the Bearer Token
The bearer token is associated with a service account. To retrieve it:

Find the service account that you intend to use (e.g., argocd-manager):
```
kubectl get serviceaccount -n <namespace>
```
Get the Service Account Secret

Service accounts are linked to secrets. Retrieve the secret name:

```
kubectl get serviceaccount <service-account-name> -n <namespace> -o yaml
```
Look for the secrets field. Copy the name of the linked secret.

Extract the Token

Run the following command to get the bearer token:
```
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.token}' | base64 --decode
```
2. Get the CA Certificate
The CA certificate is typically included in the same secret as the token. To extract it:
```
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.ca\.crt}' | base64 --decode
```

The extracted Cert and Token we will need to add it to secret in Repo ArgoCD-Infrastructure!!!!!!
