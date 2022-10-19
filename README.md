# ArgoCD apps of apps for Kubeflow

Based on kubeflow manifests version 1.5.0

Manifest are copied here, waiting migration on kustomize 4 with remote resources : https://github.com/kubeflow/manifests/issues/1797
Or waiting for https://github.com/argoproj/argo-cd/issues/677 for removing local kustomization files

## Install
add this app to argocd instance in your apps of apps :
 ```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubeflow
spec:
  destination:
    namespace: kubeflow
    server: https://kubernetes.default.svc
  project: [YOUR PROJECT]
  source:
    path: apps/
    repoURL: https://github.com/tom333/argocd-kubeflow.git
    targetRevision: HEAD
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      selfHeal: true
      allowEmpty: true
      prune: true
 ```

## Must add kustomize 3.2.0 to argocd :
Based on file values.yaml of argo helm chart :
```
  repoServer:
    volumes:
      - name: custom-tools
        emptyDir: {}
    initContainers:
      - name: download-tools
        image: alpine:3.8
        command: [sh, -c]
        args:
          - wget https://github.com/kubernetes-sigs/kustomize/releases/download/v3.2.0/kustomize_3.2.0_linux_amd64 &&
            mv kustomize_3.2.0_linux_amd64 /custom-tools/kustomize_3.2.0 &&
            chmod +x /custom-tools/kustomize_3.2.0
        volumeMounts:
          - mountPath: /custom-tools
            name: custom-tools
    volumeMounts:
      - mountPath: /usr/local/bin/kustomize_3.2.0
        name: custom-tools
        subPath: kustomize_3.2.0
  server:
    config:
      kustomize.path.v3.2.0: /usr/local/bin/kustomize_3.2.0
 ```

## Ingress for local network
 ```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  labels:
    app: kubeflow
  name: kubeflow-ingress
  namespace: istio-system
  annotations:
    cert-manager.io/cluster-issuer: "kubeflow-self-signing-issuer"
spec:
  rules:
    - host: kubeflow.internal.lan
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: istio-ingressgateway
                port:
                  number: 80
  tls:
    - hosts:
        - kubeflow.internal.lan
      secretName: kubeflow.internal.lan
 ```

## if error like https://github.com/kubeflow/manifests/issues/1931
`upstream connect error or disconnect/reset before headers. reset reason: connection failure, transport failure reason: TLS error: 268435703:SSL routines:OPENSSL_internal:WRONG_VERSION_NUMBER`


launch command :

`kubectl get deploy -n kubeflow  -o name | xargs kubectl rollout restart -n kubeflow`
