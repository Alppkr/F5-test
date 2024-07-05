Installing CIS and IPAM

First we need to complete prerequisites:
These are the mandatory requirements for deploying CIS:

    Kubernetes Cluster must be up and running.
    AS3: 3.18+ must be installed on your BIG-IP system.
    Use the latest TLS version and cipher suites in Kubernetes for kube-api.
    Create a BIG-IP partition to manage Kubernetes objects. This partition can be created either via the GUI (System > Users > Partition List) or via our TMOS CLI:
        create auth partition <cis_managed_partition>
    You need a user with administrative access to this partition.
    If you need to pull the k8s-bigip-ctlr image from a private Docker registry, store your Docker login credentials as a Secret.

Then we need to move CISInstallCommon folder and edit F5password.yaml we need to set base64encoded username and password for new created partition and install serviceaccount.yaml and F5password.yaml

We will move to CISInstall folder. There are two CISF5* yaml to for different F5. I will mention configuration parameters below:

    "--bigip-username=$(BIGIP_USERNAME)" #username for the big ip system. we have already created with F5password.yaml
    "--bigip-password=$(BIGIP_PASSWORD)", #password for the big ip system. we have already created with F5password.yaml
    "--bigip-url=10.30.1.155:8443", #internal management url for bigip
    "--bigip-partition=K8s", #new created partition for kubernetes
    "--pool-member-type=cluster", # it helps to directly communicate with pods not with kube-proxy
    "--custom-resource-mode=true", # When true ConfigMaps, Routes, and Ingress are not processed by CIS.
    "--insecure", # this enables insecure SSL communication to the BIG-IP system.
    "--ipam=true", # use to enable IPAM
    "--namespace=F5-CIS-1", #Namespace name which crd are created in for first F5
    "--namespace-label=apps", #Namespace label which appllication deployed namespaces should be label (only application namespaces label with this label not F5 namespaces)

After you completed necessary changes in the yaml files and created new namespaces for CRDs(F5-CIS-1,F5-CIS-2). CISF5* should be deployed. Then, CRD.yaml must be deployed.

Now we will move to IPAMInstall folder. There are three yaml file. F5-ipam-rbac.yaml file must be deployed first. No need to install F5-ipam-persistentvolume.yaml for EKS in dynamic provisioning mode but I put it here if you need somehow. At least, we will deploy F5-ipam-deployment.yaml but you need to set ip ranges which will be used to create virtualservers. At the beginning, IP address can be assigned to respected NICs in AWS for BIGIPs then this IP address can use below configuration parameter.

    --ip-range='{"f5-1":"10.192.75.113-10.192.75.116","f5-2":"10.192.125.30-10.192.125.50"}' # different labels must be created for different f5 subnets

After all the installation completed, we can move to deploy example application.

First we need to label namespace which we will use to deploy application(you can label default namespace). Then we will move virtualserver folder and deploy deployment.yaml Now, we deploy f5-virtualserver* yaml files. It will create virtualServer in their own namespaces. You can check the configuration details below.

    apiVersion: "cis.f5.com/v1"
    kind: VirtualServer
    metadata:
      name: foo-virtual-server
      namespace: F5-CIS-1 #BIGIP namespace it changes based on which BIGIP we want to use for first one we can set it to F5-CIS-1. It is releated with CIS deployment parameter (--namespace=F5-CIS-1).
      labels:
        f5cr: "true" # Must be set
    spec:
      # This is an insecure virtual, Please use TLSProfile to secure the virtual
      # check out tls examples to understand more.
      host: foo.example.com
      ipamLabel: "f5-1" # ipamlabel for first BIGIP. IP will assign based on ip range we set will installing IPAM controller
      virtualServerName: "foo-virtual-server" # virtual server name which will be shown in GUI
      pools:
      - path: /
        service: foo-service # name of the service we deployed in default namespace
        serviceNamespace: default # namespace of the service located
        servicePort: 80

