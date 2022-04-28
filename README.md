## ZTP Spoke cluster deployment (using assisted-service)
---

The purpose of this document is to give general knowledge and basic understanding on the OpenShift spoke cluster deployment using ZTP method.
ZTP is using **assisted service** that is deployed by Infrastructure operator and used to manage baremetal openshift installations.

For more info please refer to [https://github.com/openshift/assisted-service](https://github.com/openshift/assisted-service)

Also : https://github.com/jparrill/ztp-the-hard-way

Refer to [Acronyms and terminology](#acronyms-and-terminology) for the components names and abbreviation.


### Prerequisites

- Hub Cluster should be present
- assisted installer should be deployed on Hub Cluster as part of RHACM, MCE or/and Infrastructure operator

- Before starting the installation, check that ClusterImageSet is present. Refer to example: [ClusterImageSet.yaml](hub-example-manifests/ClusterImageSet.yaml)


### Relations diagram

See color pattern in order to see the relations between crds

![CRD relations](./relations.svg)


### Installation procedure overview


1. Each spoke cluster should be declared in different namespace on the Hub cluster

In our example we will create namespace called ***spoke-0*** for cluster called **"spoke-0"**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: "spoke-0" #the name of the namespace in which we will declare the spoke cluster
  labels:
    name: "spoke-0"
```

2. All images we need for the spoke cluster deployment are stored in the registry. It can be public public registry or local one but we need the pull secret in order to pull the image from the registry

refer to example: [02-pull-secret.yaml](spoke-0-manifests/02-pull-secret.yaml)

3. For each cluster we should have cluster definition. The declaration is done by applying the ClusterDeployment crd

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterDeployment
metadata:
  name: spoke-0
  namespace: spoke-0
spec:
  baseDomain: qe.lab.redhat.com  # domain name for spoke cluster
  clusterName: spoke-0
  clusterInstallRef:  #relation to AgentClusterInstall
    group: extensions.hive.openshift.io
    kind: AgentClusterInstall
    name: spoke-0
    version: v1beta1
  platform:
    agentBareMetal:
      agentSelector:
        matchLabels:
          bla: aaa
  pullSecretRef: #relation to pullSecret
    name: spoke-0-pull-secret
```

1. For each cluster we should provide:
   - OpenShift version to deploy
   - API VIP
   - INGRESS VIP
   - cluster network (POD IPs range)
   - service network (k8s services IPs range)
   - number of control planes (masters) and worker nodes

All this info we declare in AgentClusterInstall crd:

```yaml
apiVersion: extensions.hive.openshift.io/v1beta1
kind: AgentClusterInstall
metadata:
  name: spoke-0
  namespace: "spoke-0"
spec:
  clusterDeploymentRef:
    name: spoke-0
  imageSetRef:
    name: "4.10"   # This is the clusterImageSet name to use
  apiVIP: "192.168.128.5"  # API VIP
  ingressVIP: "192.168.128.10" #Ingress VIP
  networking:
    clusterNetwork:
    - cidr: "10.128.0.0/14" # PODS IP Ranges
      hostPrefix: 23
    serviceNetwork:
    - "172.30.0.0/16" #Service ranges
  provisionRequirements:
    controlPlaneAgents: 3  # masters count
    workerAgents: 2 # workers count
  sshPublicKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ
```


5. Apply infraenv.yml

infraenv crd is responsible for creating discovery ISO. This ISO will be used in order to boot the VMs with initial configuration for the installation agent.
Installation agent will run on the same VM as a container and will connect to assisted-service "manager" that is deployed on Hub cluster.
After the connection assisted service will orchestrate the deployment procedure.

The **"nmStateConfigLabelSelector"** spec is optional. We should have it in case of static network configuration. While configured it will reconcile the state of nmstate config CRDs in the same namespace and will create the linux network configuration files according to the NMStateConfig CRD per BMH host.

While Discovery ISO will be generated, assisted-service will inject the ISO information into the infraenv crd.


```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: spoke-0
  namespace: spoke-0
spec:
  clusterRef:
    name: spoke-0
    namespace: spoke-0
  sshAuthorizedKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB...'
  agentLabels:
    bla: "aaa"
  pullSecretRef:
    name: spoke-0-pull-secret
  nmStateConfigLabelSelector:
    matchLabels:
      nmstate_config_cluster_name: spoke-0
```

6. Apply BMH configuration per node.
This CRD actually has connection details to hosts BMC, Boot MAC address and the Openshift Cluster role of the Node that will be booted using this BMH.

For more info please refer to: https://docs.openshift.com/container-platform/4.10/scalability_and_performance/managing-bare-metal-hosts.html

```yaml
---
apiVersion: v1
data:
    password: cGFzc3dvcmQ=
    username: YWRtaW4=
kind: Secret
metadata:
    name: spoke-master-0-0-bmc-secret
    namespace: spoke-0
type: Opaque
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
    name: spoke-master-0-0-bmh
    namespace: spoke-0
    annotations:
      inspect.metal3.io: disabled
      bmac.agent-install.openshift.io/hostname: "spoke-master-0-0"
      bmac.agent-install.openshift.io/role: "master"
    labels:
      infraenvs.agent-install.openshift.io: "spoke-0"
spec:
    bmc:
      address: redfish-virtualmedia+https://192.168.128.1:8000/redfish/v1/Systems/12435194-d48a-446c-b2d6-ea4be4dfc7ab
      credentialsName: spoke-master-0-0-bmc-secret
      disableCertificateVerification: true
    bootMACAddress: 52:54:00:ee:57:10
    automatedCleaningMode: disabled
    online: true
```

#### Acronyms and terminology
---
**ZTP** stands for Zero Touch Provisioning.

**Hive** - Hive is an operator which runs as a service on top of Kubernetes/OpenShift.The Hive service can be used to provision and perform initial configuration of OpenShift clusters

[hive documentation](https://github.com/openshift/hive)

**OLM** - operator lifecycle management used to deploy operators (like yum for linux packages)

[olm documentation](https://docs.openshift.com/container-platform/4.10/operators/understanding/olm/olm-understanding-olm.html)

**BMC** - Baseboard Management Controller (BMC) is a small, specialized processor used for remote monitoring and management of a host system.
Host can be managed thru BMC in different ways. As example thru GUI or using Redfish API.


**Redfish** - REST API used for platform management.

**Sushi Tools** - Simulation tools for Redfish protocol implementations. In our case used to manage VMs created by libvirt using RedFish protocol.

[Sushi tools documentation](https://docs.openstack.org/sushy-tools/latest)

**BMH** - Bare metal host CRD. see[ <b>metal3</b>](#metal3)

**metal3** -
<a name="metal3">
  The Bare Metal Operator implements a Kubernetes API for managing bare metal hosts(BMH).
</a>

[metal3 documentation](https://github.com/metal3-io/baremetal-operator)

**RHOS** - RedHat OS 

















https://github.com/openshift/assisted-installer-agent/tree/master/src