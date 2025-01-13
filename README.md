# RDMA: Shared, Hostdevice, Legacy SRIOV

In a previous blog we discussed how to configure RDMA on OpenShift in three distinct methods: RDMA shared, host device and legacy SRIOV.   However one of the biggest questions coming out of that blog was how do I know which one to choose?  To answer this question comprehensively we probably should first step back and discuss RDMA and the three methods in detail.

## What is RDMA?

Remote direct memory access (RDMA) is a technology, originally developed in the 1990s, that allows computers to directly access each others memory without the involvement of the hosts central processor unit (CPU) or operating system(OS).  RDMA is an extension of direct memory access(DMA) which allows direct access to a hosts memory without the use of CPU.  RDMA itself is geared toward high bandwidth and low latency applications making it a valuable component in the AI space.

NVIDIA offers GPUDirect RDMA which is a technology that provides a direct data path between the GPU memory directly between two or more hosts leveraging the NVIDIA networking device.  This configuration provides a significant decrease in latency and offloads the CPU of the hosts.  When leveraging this technology from NVDIA the consumer has the ability to configure it multiple ways to leverage the underlying technology but also based on the consumers use cases.

The three configuration methods for GPUDirect RDMA are as follows:

* RDMA Shared Device
* RDMA SR-IOV Legacy Device
* RDMA Host Device

Let's take a look at each of these options and discuss why one might be used over the other depending on a consumers use case.

## RDMA Shared Device

When using the NVIDIA network operator in OpenShift there is a configuration method in the NicClusterPolicy called RDMA shared device.  This method allows for an RDMA device to be shared among multiple pods on the OpenShift worker node where the device is exposed.  The user defined networks of those pods use VXLAN or VETH networking devices inside OpenShift.   Usually those devices are defined in the NicClusterPolicy by specifying the physical device name like in the code snippet below:

~~~bash
  rdmaSharedDevicePlugin:
    config: |
      {
        "configList": [
          {
            "resourceName": "rdma_shared_device_ib",
            "rdmaHcaMax": 63,
            "selectors": {
              "ifNames": ["ibs2f0"]
            }
          },
          {
            "resourceName": "rdma_shared_device_eth",
            "rdmaHcaMax": 63,
            "selectors": {
              "ifNames": ["ens8f0np0"]
            }
          }
        ]
      }
~~~

The example above shows both an RDMA shared device for an ethernet interface and an infiniband interface.   We also define the number of pods that could consume the interface via the rdmaHcaMax parameter.   In the NicClusterPolicy we can define as many interfaces that we have in the worker nodes.  

In an RDMA shared device configuration keep in mind that the pods sharing the device will be competing for the bandwidth and latency of the same device as with any shared resource.  Thus an RDMA shared device is better suited for developer environments where performance and latency are not key but the ability to test RDMA functionality across nodes is important.

## RDMA SR-IOV Legacy Device

Single Root IO Virtualization (SR-IOV) specification is a standard for a type of PCI device assignment that, like an RDMA shared device, can share a single device with multiple pods.  However the way the device is shared is very different because SR-IOV can segment the compliant network device at the hardware layer.  The network device is recognized on the node as a physical function (PF) and when segmented creates multiple virtual functions (VFs).  Each VF can be used like any other network device.  The SR-IOV network device driver for the device determines how the VF is exposed in the container:
netdevice driver: A regular kernel network device in the netns of the container
vfio-pci driver: A character device mounted in the container
Unlike a shared device though an SR-IOV device can only be shared with the number of pods based off the number of VFs the device supports.  However since each VF is like having direct access to the device the performance is ideal for workloads that are latency and bandwidth sensitive.

The configuration of the SRI-IOV devices doesn't take place in the NVIDIA network operator NicClusterPolicy, though we still need the policy for the driver, but rather in the SriovNetworkNodePolicy of the worker node.   The below example shows how we define a vendor and pfName for the nicSelector along with a numVfs which defines the number of VFs to create.  

~~~bash
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: sriov-legacy-policy
  namespace:  openshift-sriov-network-operator
spec:
  deviceType: netdevice
  mtu: 1500
  nicSelector:
    vendor: "15b3"
    pfNames: ["ens8f0np0#0-7"]
  nodeSelector:
    feature.node.kubernetes.io/pci-15b3.present: "true"
  numVfs: 8
  priority: 90
  isRdma: true
  resourceName: sriovlegacy
~~~

Once the configuration is in place RDMA SR-IOV workloads that require high bandwidth and low latency are great candidates for this type of configuration where multiple pods need that performance from a single network device.

# RDMA Host Device

Host device is in some ways a lot like SR-IOV in that a host device creates an additional network on a pod allowing direct physical ethernet access on the worker node.  The plugin allows the network device to be moved from the hosts network namespace to the pods network namespace.  However unlike SR-IOV once the device is passed into a pod the device is not available to any other host until the pod that is using it is removed from the system which makes it far more restrictive.

The configuration of this type of RDMA is handled again through the NVIDIA network operator NicClusterPolicy.   The irony here is even though it is not an SR-IOV configuration the DOCA driver uses the SRIOV network device plugin to do the device passing.   Below is an example of how to configure this type of RDMA where we will set a resourceName and use the NVIDIA vendors selector and any device that has the RDMA capability to be exposed as a host device.  If there are multiple cards in the system the configuration will expose all of them assuming they match the vendor id and have RDMA capabilities.

~~~bash
  sriovDevicePlugin:
      image: sriov-network-device-plugin
      repository: ghcr.io/k8snetworkplumbingwg
      version: v3.7.0
      config: |
        {
          "resourceList": [
              {
                  "resourcePrefix": "nvidia.com",
                  "resourceName": "hostdev",
                  "selectors": {
                      "vendors": ["15b3"],
                      "isRdma": true
                  }
              }
          ]
        }
~~~

The RDMA host device is normally leveraged where the other two options above are not feasible.  For example the use case requires performance but other requirements don't allow for the use of VFs.  Maybe the cards themselves do not support SR-IOV, or there is not enough PCI express base address registers(BAR) or maybe the system board does not support SR-IOV.   There are also rare cases where the SR-IOV netdevice driver does not support all the capabilities of the network device compared to the PF driver and the workload requires those features.

As we have discussed this blog covered what RDMA is and how one can configure three different methods of RDMA with the NVIDIA network operator.   We also discussed and compared why one might use one method over the other along the way.   Hopefully this gives those looking to adopt this technology enough detail to pursue the right solution for their use case.
