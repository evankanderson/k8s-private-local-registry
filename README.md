# Private local registry for Kubernetes clusters (POC)

This is a configuration for a running a registry on a Kubernetes cluster, _which is trusted by the cluster's `kubelets`_. This simplifes local and on-cluster build and test **for ephemeral testing environments**. This setup makes minimal efforts at long-term persistence, and is **not intended for production environments**.

## Prerequisites

- [cert-manager](https://cert-manager.io) installed on the cluster
- A [default StorageClass](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class-1) configured on the cluster
- A cluster which can produce IP addresses for LoadBalancers. The current script uses the loadbalancer's IP to configure DNS using the `nip.io` service.
- A [default IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#default-ingress-class) configured on the cluster. This probably involves installing an Ingress implementation.

- Permissions to run privileged pods with the ability to mount host filesystems and network namespaces

## How it works

1. First, the system provisions a self-signed certificate for use by the registry from cert-manager.

   ```bash
   kubectl apply -f certificate.yaml
   ```

2. Next (in the same namespace), we provision a Deployment with a single replica to run the Docker Registry. This registry is backed by a PersistentVolumeClaim, so there is some durability across restarts. We also provision an Ingress with a fake name to provide access external access to the Registry.

   ```bash
   kubectl apply -f registry
   ```

3. Next, we read the provisioned LoadBalancer IP from the Ingress object, and update the Ingress object to use an `nip.io` DNS name to provision and serve using a self-signed certificate. We also store the registered ingress name in a ConfigMap for use by the next stage.

   ```bash
   kubectl apply -f update-ingress.yaml
   ```

4. Lastly, we run a privileged DaemonSet to inject the created certificate into the CRI trust store on each node. This uses a privileged Pod which can mount the correct _host_ directories, as well as connect to `systemd` to restart the `containerd` process if you're using that CRI. (Docker and CRI-O do not require a configuration restart; newer versions of containerd also do not.)

   ```bash
   kubectl apply -f registry-trust.yaml
   ```

## Exposing the registry outside the cluster

Because step 2 and 3 provisions an Ingress with an externally-visible IP address, you should be able to access the registry from outside the cluster if you download the CA certificate. If you have the `kube-view-secret` plugin installed, fetching the CA certificate is as easy as:

   ```bash
   kubectl view secret -n registry ingress-cert ca.crt > ca.crt
   ```

Using it is somewhat trickier. The registry's URL will be `registry.$IP.nip.io`, or you can get it from `kubectl get ingress -n registry local`.

### Docker Desktop

Create a directory `$HOME/.docker/certs.d/$REGISTRY`, and copy the `ca.crt` into it, then **restart docker**.

### Other tools like `crane` or `ko`

On Mac, you'll need to add the `ca.crt` to the system keychain, with an invocation like:

   ```bash
   security add-trusted-cert -d -r trustRoot -k ~/Library/Keychains/login.keychain ca.crt
   ```

On Linux, you can set the `$SSL_CERT_DIR` directory to point to the directory your `ca.crt` file is in.

On Windows, you'll need to use `mmc` (Microsoft Management Console), I think. I haven't tried this.