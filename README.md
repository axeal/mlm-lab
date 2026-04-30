# mlm-lab

## Prerequisites

- An RKE2 cluster with the [Traefik Ingress Controller](https://docs.rke2.io/networking/networking_services#ingress-controller)
- `helm` ([Helm installation documentation](https://helm.sh/docs/intro/install/#from-script))
- `kubectl` (`alias kubectl=/var/lib/rancher/rke2/bin/kubectl` on RKE2 server node)
- A kubeconfig with admin permissions (`export KUBECONFIG=/etc/rancher/rke2/rke2.yaml` on RKE2 server node)

## RKE2 cluster configuration

A simple single node RKE2 cluster can be provisioned by following the [RKE2 Quick Start documentation](https://docs.rke2.io/install/quickstart).

The only cluster configuration that needs to be set is the choice of Traefik as the Ingress Controller.

Create the file `/etc/rancher/rke2/config.yaml` with the following contents, before installing RKE2:
```
ingress-controller: traefik
```

## Uyuni installation

1. Install [cert-manager](https://cert-manager.io/docs/installation/helm/)
    ```
    helm upgrade --install \
        cert-manager oci://quay.io/jetstack/charts/cert-manager \
        --namespace cert-manager \
        --create-namespace \
        --set crds.enabled=true
    ```

1. Install [trust-manager](https://cert-manager.io/docs/trust/trust-manager/installation/#3-install-trust-manager)
    ```
    helm upgrade --install \
        trust-manager oci://quay.io/jetstack/charts/trust-manager \
        --namespace cert-manager
    ```

1. Install [local-path-provisioner](https://github.com/rancher/local-path-provisioner#installation)
    ```
    kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.35/deploy/local-path-storage.yaml
    ```

1. Make local-path the default StorageClass
    ```
    kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

1. Configure Traefik according to [the uyuni-charts documentation](https://github.com/uyuni-project/uyuni-charts/blob/main/README.md#uyuni-server-configuration).

    Create the file `/var/lib/rancher/rke2/server/manifests/rke2-traefik-config.yaml` on the RKE2 server node with the following [HelmChartConfig](https://docs.rke2.io/add-ons/helm#customizing-packaged-components-with-helmchartconfig) manifest:
    ```
    apiVersion: helm.cattle.io/v1
    kind: HelmChartConfig
    metadata:
    name: rke2-traefik
    namespace: kube-system
    spec:
    valuesContent: |-
        ports:
        reportdb-pgsql:
            port: 5432
            expose:
            default: true
            exposedPort: 5432
            protocol: TCP
            hostPort: 5432
            containerPort: 5432
        salt-publish:
            port: 4505
            expose:
            default: true
            exposedPort: 4505
            protocol: TCP
            hostPort: 4505
            containerPort: 4505
        salt-request:
            port: 4506
            expose:
            default: true
            exposedPort: 4506
            protocol: TCP
            hostPort: 4506
            containerPort: 4506
    ```

1. Clone uyuni-charts GitHub repository
    ```
    git clone https://github.com/uyuni-project/uyuni-charts.git
    ```

1. Populate values file

    Create a `values.yaml` file with the following contents, and set `fqdn` to the value of the hostname for uyuni. You can use a [nip.io](https://nip.io/) address, pointing to the external IP of an RKE2 cluster node.
    ```
    credentials:
    db:
        admin:
        password: admin
        internal:
        password: admin
        reportdb:
        password: admin
    admin:
        password: admin

    global:
        fqdn: "your.fq.dn"

    server-helm:
    # All the values for server-helm should follow
    repository: registry.opensuse.org/systemsmanagement/uyuni/master/containerfile/uyuni
    ingress:
        type: traefik
        class: traefik
    # https://github.com/uyuni-project/uyuni/tree/master/containers/server-helm#apparmor
    server:
        superPrivileged: true
    tftp:
        enabled: false
    ```

1. Update server-helm chart version
    1. Query the latest version of the server-helm chart
        ```
        helm show chart oci://registry.opensuse.org/systemsmanagement/uyuni/master/charts/uyuni/server-helm
        ```
    2. Update the server-helm chart version in `uyuni-charts/server-selfsigned/Chart.yaml` to reflect the latest version, eg. `2026.4.0`.

1. Build helm dependencies
    ```
    helm dependencies build uyuni-charts/server-selfsigned
    ```

1. Install uyuni helm chart

    ```
    helm upgrade --install \
        uyuni uyuni-charts/server-selfsigned \
        --namespace uyuni \
        --create-namespace \
        -f values.yaml
    ```
