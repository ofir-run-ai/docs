# Scheduling Virtual Machines using Run:AI

Many organizations  use virtual machines (VMs) to provide operating system abstraction to users. Containers are different than VMs but serve a similar purpose. Containers at large scale are best managed by Kubernetes and Run:AI is based on Kubernetes. 

It is possible to mix and match containers and VMs to some extent using a technology called [KubeVirt]( https://kubevirt.io){target=_blank}. Kubevirt allows running VMs inside containers on top of Kubernetes. 

This article describes how to use Kubevirt to schedule VMs with GPUs.

## Limitations

At this point, Run:AI requires that all GPUs in the cluster will be used for VMs and not containers. This limitation will be lifted soon. Check with Run:AI support for up to date information. 

## Preparations

Making GPUs visible to VMs is not trivial. It requires either a license for NVIDIA software called [NVIDIA vGPU](https://www.nvidia.com/en-us/data-center/virtual-solutions/){target=_blank} or explicit mapping of the GPU devices to the virtual machines. This guide relates to the later option. 

### Install KubeVirt

Install KubeVirt using the following [guide](https://kubevirt.io/quickstart_cloud/){target=_blank}.

### Assign host devices to virtual machines

For each node in the cluster that we want to use with VMs we must:

* Identify all GPU cards.
* Map GPU cards for KubeVirt to pick up (called _assigning host devices to a virtual machine_).

Instructions for identifying GPU cards are operating-system specific. For Ubuntu 20.04 run:

``` bash
lspci -nnk -d 10de:
```

Search for GPU cards that are marked with the text _Kernel driver in use_. Save the PCI Address, for example: __10de:1e04__

!!! Important
    Once exposed, these GPUs cannot be used by regular pods. Only VMs. 




To map GPUs to KubeVirt follow the instructions [here](https://kubevirt.io/user-guide/virtual_machines/host-devices/){target=_blanks}. Specifically, run:


```
kubectl edit  kubevirt -n kubevirt -o yaml
```

And add all of the PCI Addresses of all GPUs of all Nodes concetenated by commas:

``` YAML
spec:
  certificateRotateStrategy: {}
  configuration:
    developerConfiguration:
      featureGates:
      - GPU
      - HostDevices
    permittedHostDevices:
      pciHostDevices:
      - pciVendorSelector: <PCI-ADDRESS>,<PCI-ADDRESS>,
        resourceName: nvidia.com/gpu
```

## Assign GPUs to VMs

You must create a CRD called _vm_ for each virtual machine. `vm` is a reference to a virtual machine and its capabilities.

The Run:AI project is matched to a Kubernetes namespace. Unless manually configured, the namespace is `runai-<PROJECT-NAME>`. __Per Run:AI Project__, create a `vm` object. See KubeVirt [documentation](https://kubevirt.io/labs/kubernetes/lab1) example. Specifically, the created YAML should look like:


``` YAML
spec:
  running: false
  template:
    metadata:
      creationTimestamp: null
      labels:
....
        priorityClassName: <WORKLOAD-TYPE>
        project: <PROJECT-NAME>
    spec:
      schedulerName: runai-scheduler
      domain:
        devices:
          gpus:
          - deviceName: nvidia.com/gpu # identical name to resourceName above
            name: gpu1  # name here is arbitrary and is not used. 

```

Where `<WORKLOAD-TYPE>` is `train` or `build`

????  SHOULD this BE VM template.  does it make sense to embed the workload-type in the vm


## Start a VM

Run:

```
virtctl start testvm -n runai-test
```

You can now see the VMs pod in Run:AI:

```
runai list -A
NAME    STATUS   AGE  NODE         IMAGE                                   TYPE  PROJECT  USER  GPUs Allocated (Requested)  PODs Running (Pending)  SERVICE URL(S)
testvm  Running  0s   master-node  quay.io/kubevirt/virt-launcher:v0.47.1        test           1 (1)                       1 (0)
```


??? Does it show on UI?

