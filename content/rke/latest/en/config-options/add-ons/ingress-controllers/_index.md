---
title: K8s Ingress Controllers
description: By default, RKE deploys the NGINX ingress controller. Learn how to schedule and disable default k8s ingress controllers, and how to configure NGINX controller
weight: 262
---

By default, RKE deploys the NGINX ingress controller on all schedulable nodes.

> **Note:** As of v0.1.8, only workers are considered schedulable nodes, but prior to v0.1.8, worker and controlplane nodes were considered schedulable nodes.  

RKE will deploy the ingress controller as a DaemonSet with `hostnetwork: true`, so ports `80`, and `443` will be opened on each node where the controller is deployed.

The images used for ingress controller is under the [`system_images` directive]({{<baseurl>}}/rke/latest/en/config-options/system-images/). For each Kubernetes version, there are default images associated with the ingress controller, but these can be overridden by changing the image tag in `system_images`.

## Scheduling Ingress Controllers

If you only wanted ingress controllers to be deployed on specific nodes, you can set a `node_selector` for the ingress. The label in the `node_selector` would need to match the label on the nodes for the ingress controller to be deployed.

```yaml
nodes:
- address: 1.1.1.1
  role: [controlplane,worker,etcd]
  user: root
  labels:
    app: ingress

ingress:
  provider: nginx
  node_selector:
    app: ingress
```

## Tolerations

_Available as of v1.2.4_

The configured tolerations apply to the `default-http-backend` Deployment.

```
ingress:
  tolerations:
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationseconds: 300
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationseconds: 300
```

To check for applied tolerations `default-http-backend` Deployment, use the following commands:

```
kubectl -n ingress-nginx get deploy default-http-backend -o jsonpath='{.spec.template.spec.tolerations}'
```

## Disabling the Default Ingress Controller

You can disable the default controller by specifying `none` to  the ingress `provider` directive in the cluster configuration.

```yaml
ingress:
    provider: none
```
## Configuring NGINX Ingress Controller

For the configuration of NGINX, there are configuration options available in Kubernetes. There are a [list of options for the NGINX config map](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/configmap.md) , [command line extra_args](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/cli-arguments.md) and [annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/).

```yaml
ingress:
  provider: nginx
  options:
    map-hash-bucket-size: "128"
    ssl-protocols: SSLv2
  extra_args:
    enable-ssl-passthrough: ""
```

### Disabling NGINX Ingress Default Backend

As of v0.20.0, you can disable the [default backend service](https://kubernetes.github.io/ingress-nginx/user-guide/default-backend/) for the ingress controller. This is possible because `ingress-nginx` will fall back to a local 404 page, and does not require a backend service. The service can be enabled/disabled with a boolean value.

```yaml
ingress:
  default_backend: false
```

> **What happens if the field is omitted?** The value of `default_backend` will default to `true`. This maintains behavior with older versions of `rke`. However, a future version of `rke` will change the default value to `false`.

## Configuring an NGINX Default Certificate

When configuring an ingress object with TLS termination, you must provide it with a certificate used for encryption/decryption. Instead of explicitly defining a certificate each time you configure an ingress, you can set up a custom certificate that's used by default.

Setting up a default certificate is especially helpful in environments where a wildcard certificate is used, as the certificate can be applied in multiple subdomains.

>**Prerequisites:**
>
>- Access to the `cluster.yml` used to create the cluster.
>- The PEM encoded certificate you will use as the default certificate.

1. Obtain or generate your certificate key pair in a PEM encoded form.

2. Generate a Kubernetes secret from your PEM encoded certificate with the following command, substituting your certificate for `mycert.cert` and `mycert.key`.

    ```
    kubectl -n ingress-nginx create secret tls ingress-default-cert --cert=mycert.cert --key=mycert.key -o yaml --dry-run=true > ingress-default-cert.yaml
    ```
3. Include the contents of `ingress-default-cert.yml` inline with your RKE `cluster.yml` file. For example:

    ```yaml
    addons: |-
      ---
      apiVersion: v1
      data:
        tls.crt: [ENCODED CERT]
        tls.key: [ENCODED KEY]
      kind: Secret
      metadata:
        creationTimestamp: null
        name: ingress-default-cert
        namespace: ingress-nginx
      type: kubernetes.io/tls
    ```
4. Define your ingress resource with the following `default-ssl-certificate` argument, which references the secret we created earlier under `extra_args` in your `cluster.yml`:

    ```yaml
    ingress: 
      provider: "nginx"
      extra_args:
        default-ssl-certificate: "ingress-nginx/ingress-default-cert"
    ```

5. **Optional:** If you want to apply the default certificate to ingresses in a cluster that already exists, you must delete the NGINX ingress controller pods to have Kubernetes schedule new pods with the newly configured `extra_args`.

    ```
    kubectl delete pod -l app=ingress-nginx -n ingress-nginx
    ```
