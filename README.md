# anthos-sr-iov-setup

1. Customer to install drivers for Mellanox
2. Execute the following command for each interface (2 x per node)

```
ls -all /sys/class/net/ |grep ens192
lrwxrwxrwx.  1 root root 0 Sep 16 11:12 ens192 -> ../../devices/pci0000:00/0000:00:16.0/0000:0b:00.0/net/ens192
```

3. Execute the following command for each interface (2 x per node)

```
find /sys/devices -name sriov_numvfs | grep 0000:0b:00.0/sriov_numvfs
/sys/devices/pci0000:00/0000:00:1f.6
```

4. Execute the following command for each interface (2 x per node) to create 10 VFs on the PF

```
echo 10 > /sys/devices/pci0000:00/0000:00:1f.6/sriov_numvfs
```

5. Execute the following command for on each node to capture the PCI vendor and device codes

```
lspci -nn | grep Eth
00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (2) I219-LM [8086:15b7]
```

6. Example configmap that needs to be populated and applied with “resourceName”, “resourcePrefix”,“vendors:”, “devices” and “pfNames” values.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList": [
            {
                "resourceName": "vlan_a",
                "resourcePrefix": "gke",
                "selectors": {
                    "vendors": ["15b3"],
                    "devices": ["1018"],
                    "pfNames": ["enp0s0f0"]
                }
            },
            {
                "resourceName": "vlan_b",
                "resourcePrefix": "gke",
                "selectors": {
                    "vendors": ["15b3"],
                    "devices": ["1018"],
                    "pfNames": ["enp0s0f1"]
                }
            }
        ]
    }
```

7. Then you should see the gke/sriov resource on your specific nodes. For example:

```
$ kubectl --kubeconfig=bmctl-workspace/cluster4/cluster4-kubeconfig get nodes smf-bm-01 -o yaml
apiVersion: v1
kind: Node
metadata:
...
status:
  addresses:
  - address: 192.168.1.32
    type: InternalIP
  - address: smf-bm-01
    type: Hostname
  allocatable:
    cpu: "45"
    ephemeral-storage: "210725550141"
    gke/vlan_a: "20"
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 64627876Ki
    pods: "110"
  capacity:
    cpu: "48"
    ephemeral-storage: 228651856Ki
    gke/vlan_a: "20"
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 65516708Ki
    pods: "110"
```

8. You can create a normal pod to verify the SRIOV installation.  First create a NetworkAttachmentDefinition that informs multus how to configure the 2nd interface.

```
cat <<EOF | kubectl --kubeconfig=bmctl-workspace/cluster4/cluster4-kubeconfig apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  annotations:
    k8s.v1.cni.cncf.io/resourceName: gke/sriov_iavf
  name: net-sriov-vlan-a
spec:
  config: |
    {
        "type": "sriov",
        "cniVersion": "0.3.1",
        "LogFile": "/var/log/net-sriov-vlan-a-cni.log",
        "LogLevel": "debug",
        "vlan": 1411,
        "name": "sriov-network",
        "ipam": {
          "type": "whereabouts",
          "range": "192.168.2.0/24",
          "range_start": "192.168.2.128",
          "range_end": "192.168.2.255"
        }
    }
EOF
```

9. You can create the pod with:

```
cat <<EOF | kubectl --kubeconfig=bmctl-workspace/cluster1/cluster1-kubeconfig apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dnsutils
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dnsutils
  Ttemplate:                                                                                  
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: net-sriov-vlan-a
      labels:
        app: dnsutils
    spec:
      containers:
      - name: dnsutils
        resources:
          limits:
            gke/vlan_a: "1"
          requests:
            gke/vlan_a: "1"
        image: gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
        command:
          - sleep
          - "3600"
        imagePullPolicy: IfNotPresent
      restartPolicy: Always
EOF
```
