
# Installing CIS and IPAM

First, we need to complete the prerequisites:
These are the mandatory requirements for deploying CIS:

1. The Kubernetes Cluster must be up and running.
2. AS3 3.18+ must be installed on your BIG-IP system.
3. Use the latest TLS version and cipher suites in Kubernetes for kube-api.
4. Create a BIG-IP partition to manage Kubernetes objects. This partition can be created either via the GUI (System > Users > Partition List) or via the TMOS CLI:
    ```
    create auth partition <cis_managed_partition>
    ```
5. You need a user with administrative access to this partition.
6. If you need to pull the k8s-bigip-ctlr image from a private Docker registry, store your Docker login credentials as a Secret.

Next, we need to move to the CISInstallCommon folder and edit the `F5password.yaml` file. We need to set the base64-encoded username and password for the newly created partition and install `serviceaccount.yaml` and `F5password.yaml`.

We will move to the CISInstall folder. There are two `CISF5*` yaml files for different F5 configurations. I will mention the configuration parameters below:

```
"--bigip-username=$(BIGIP_USERNAME)" # Username for the BIG-IP system, already created with F5password.yaml
"--bigip-password=$(BIGIP_PASSWORD)" # Password for the BIG-IP system, already created with F5password.yaml
"--bigip-url=10.30.1.155:8443"       # Internal management URL for BIG-IP
"--bigip-partition=K8s"              # Newly created partition for Kubernetes
"--pool-member-type=cluster"         # Enables direct communication with pods, bypassing kube-proxy
"--custom-resource-mode=true"        # When true, ConfigMaps, Routes, and Ingress are not processed by CIS
"--insecure"                         # Enables insecure SSL communication to the BIG-IP system
"--ipam=true"                        # Enables IPAM
"--namespace=F5-CIS-1"               # Namespace name where CRDs are created for the first F5
"--namespace-label=apps"             # Namespace label for application-deployed namespaces (only application namespaces labeled with this label, not F5 namespaces)
```

After you have made the necessary changes to the YAML files and created new namespaces for CRDs (`F5-CIS-1`, `F5-CIS-2`), the `CISF5*` should be deployed. Then, `CRD.yaml` must be deployed.

Now, we will move to the IPAMInstall folder. There are three YAML files. The `F5-ipam-rbac.yaml` file must be deployed first. There's no need to install `F5-ipam-persistentvolume.yaml` for EKS in dynamic provisioning mode, but it's included if needed. Finally, we will deploy `F5-ipam-deployment.yaml`, but you need to set the IP ranges which will be used to create virtual servers. Initially, IP addresses can be assigned to the respective NICs in AWS for BIG-IPs, and then this IP address can be used in the configuration parameter below:

```
--ip-range='{"f5-1":"10.192.75.113-10.192.75.116","f5-2":"10.192.125.30-10.192.125.50"}' # Different labels must be created for different F5 subnets
```

After completing all the installations, we can move to deploy an example application.

First, we need to label the namespace where we will deploy the application (you can label the default namespace). Then, we will move to the virtualserver folder and deploy `deployment.yaml`. Next, we deploy the `f5-virtualserver*` YAML files. These files will create a VirtualServer in their respective namespaces. You can check the configuration details below:

```
apiVersion: "cis.f5.com/v1"
kind: VirtualServer
metadata:
  name: foo-virtual-server
  namespace: F5-CIS-1 # BIG-IP namespace; it changes based on which BIG-IP we want to use. For the first one, set it to F5-CIS-1. This is related to the CIS deployment parameter (--namespace=F5-CIS-1).
  labels:
    f5cr: "true" # Must be set
spec:
  # This is an insecure virtual server. Please use TLSProfile to secure it.
  # Check out TLS examples to understand more.
  host: foo.example.com
  ipamLabel: "f5-1" # IPAM label for the first BIG-IP. The IP will be assigned based on the IP range set during IPAM controller installation.
  virtualServerName: "foo-virtual-server" # Virtual server name as shown in the GUI
  pools:
  - path: /
    service: foo-service # Name of the service deployed in the default namespace
    serviceNamespace: default # Namespace where the service is located
    servicePort: 80
```
