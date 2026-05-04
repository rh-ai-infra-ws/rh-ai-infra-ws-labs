# Enabling Accelerators

Enabling the utilization of these accelerators within a containerized environment, particularly on a platform like Kubernetes or OpenShift, requires a set of foundational components working in concert. The main components for enabling accelerators include:

- Node Feature Discovery (NFD): This component is vital for the initial step of resource identification. NFD detects hardware features on cluster nodes, such as the presence and model of a GPU, and then labels nodes with specific attributes (e.g., feature.node.kubernetes.io/pci-10de.present: true for an NVIDIA device). These labels are crucial for the Kubernetes scheduler to correctly match AI workloads to nodes possessing the necessary hardware.
Kubernetes Machine Management (KMM) / Hardware Driver Management: Before a device can be used, the operating system kernel needs the correct driver. While not explicitly mentioned, KMM or similar mechanisms, often working in conjunction with a dedicated driver operator, ensure that the appropriate kernel modules (drivers) are available and loaded on the nodes where accelerators reside.

- The GPU Provider Operator (e.g., NVIDIA GPU Operator): This operator plays a central management role. It is responsible for automating the lifecycle of the necessary software stack, including the deployment of device drivers and the associated device plugins. The GPU Provider Operator manages hardware drivers and deploys device plugins which are necessary for the Kubernetes kubelet to expose the hardware resources (like [nvidia.com/gpu](https://nvidia.com/gpu)) to the container runtime and, consequently, to user pods. This operator simplifies driver installation, configuration, and kernel module management.

## Node Feature Discovery

The Node Feature Discovery (NFD) Operator orchestrates all resources needed to run the NFD daemon set. As a cluster administrator, you can install the NFD Operator by using the following procedure:

1. Create a namespace for the NFD Operator

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nfd
  labels:
    name: openshift-nfd
    openshift.io/cluster-monitoring: "true"
EOF
```

2. Install the NFD OperatorGroup in the namespace you created in the previous step by creating the following objects:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: openshift-nfd-
  name: openshift-nfd
  namespace: openshift-nfd
spec:
  targetNamespaces:
  - openshift-nfd
EOF
```

3. Create the following Subscription CR:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfd
  namespace: openshift-nfd
spec:
  channel: "stable"
  installPlanApproval: Automatic
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

4. To verify that the Operator deployment is successful, run:

```bash
oc get pods -n openshift-nfd
```

?> A successful deployment shows a Running status

```text
NAME                                      READY   STATUS    RESTARTS   AGE
nfd-controller-manager-7c64df65f4-5mmm6   1/1     Running   0          85s
```

5. Create a NodeFeatureDiscovery CR:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec:
  workerConfig:
    configData: |
      core:
      #  labelWhiteList:
      #  noPublish: false
        sleepInterval: 60s
      #  sources: [all]
      #  klog:
      #    addDirHeader: false
      #    alsologtostderr: false
      #    logBacktraceAt:
      #    logtostderr: true
      #    skipHeaders: false
      #    stderrthreshold: 2
      #    v: 0
      #    vmodule:
      ##   NOTE: the following options are not dynamically run-time 
      ##          configurable and require a nfd-worker restart to take effect
      ##          after being changed
      #    logDir:
      #    logFile:
      #    logFileMaxSize: 1800
      #    skipLogHeaders: false
      sources:
      #  cpu:
      #    cpuid:
      ##     NOTE: whitelist has priority over blacklist
      #      attributeBlacklist:
      #        - "BMI1"
      #        - "BMI2"
      #        - "CLMUL"
      #        - "CMOV"
      #        - "CX16"
      #        - "ERMS"
      #        - "F16C"
      #        - "HTT"
      #        - "LZCNT"
      #        - "MMX"
      #        - "MMXEXT"
      #        - "NX"
      #        - "POPCNT"
      #        - "RDRAND"
      #        - "RDSEED"
      #        - "RDTSCP"
      #        - "SGX"
      #        - "SSE"
      #        - "SSE2"
      #        - "SSE3"
      #        - "SSE4.1"
      #        - "SSE4.2"
      #        - "SSSE3"
      #      attributeWhitelist:
      #  kernel:
      #    kconfigFile: "/path/to/kconfig"
      #    configOpts:
      #      - "NO_HZ"
      #      - "X86"
      #      - "DMI"
        pci:
          deviceClassWhitelist:
            - "0200"
            - "03"
            - "12"
          deviceLabelFields:
      #      - "class"
            - "vendor"
      #      - "device"
      #      - "subsystem_vendor"
      #      - "subsystem_device"
      #  usb:
      #    deviceClassWhitelist:
      #      - "0e"
      #      - "ef"
      #      - "fe"
      #      - "ff"
      #    deviceLabelFields:
      #      - "class"
      #      - "vendor"
      #      - "device"
      #  custom:
      #    - name: "my.kernel.feature"
      #      matchOn:
      #        - loadedKMod: ["example_kmod1", "example_kmod2"]
      #    - name: "my.pci.feature"
      #      matchOn:
      #        - pciId:
      #            class: ["0200"]
      #            vendor: ["15b3"]
      #            device: ["1014", "1017"]
      #        - pciId :
      #            vendor: ["8086"]
      #            device: ["1000", "1100"]
      #    - name: "my.usb.feature"
      #      matchOn:
      #        - usbId:
      #          class: ["ff"]
      #          vendor: ["03e7"]
      #          device: ["2485"]
      #        - usbId:
      #          class: ["fe"]
      #          vendor: ["1a6e"]
      #          device: ["089a"]
      #    - name: "my.combined.feature"
      #      matchOn:
      #        - pciId:
      #            vendor: ["15b3"]
      #            device: ["1014", "1017"]
      #          loadedKMod : ["vendor_kmod1", "vendor_kmod2"]
  operand:
    imagePullPolicy: IfNotPresent
    servicePort: 12000
  customConfig:
    configData: |
      #    - name: "more.kernel.features"
      #      matchOn:
      #      - loadedKMod: ["example_kmod3"]
      #    - name: "more.features.by.nodename"
      #      value: customValue
      #      matchOn:
      #      - nodename: ["special-.*-node-.*"]
EOF
```

6. Check that the NodeFeatureDiscovery CR was created by running the following command:

```bash
oc get pods -n openshift-nfd
```

?> A successful deployment shows a Running status

```text
NAME                                      READY   STATUS    RESTARTS   AGE
nfd-controller-manager-7c64df65f4-5mmm6   1/1     Running   0          14m
nfd-gc-67d58d4d5d-z9qtp                   1/1     Running   0          2m
nfd-master-b697bbdb8-pcgfl                1/1     Running   0          2m
nfd-worker-csmds                          1/1     Running   0          2m
```

7. Review the controller-manager pod logs:

```bash
oc logs -n openshift-nfd -l control-plane=controller-manager -t 1000
```

?> A successful deployment shows similar logs like that

```text
I0504 13:53:12.044142       1 main.go:187] "starting manager" logger="nfd.setup"
I0504 13:53:12.044245       1 server.go:50] "starting server" logger="nfd" kind="health probe" addr="[::]:8082"
I0504 13:53:12.044290       1 server.go:185] "Starting metrics server" logger="nfd.controller-runtime.metrics"
I0504 13:53:12.044309       1 server.go:50] "starting server" logger="nfd" kind="health probe" addr="[::]:8081"
I0504 13:53:12.044463       1 controller.go:178] "Starting EventSource" logger="nfd" controller="nodefeaturerule" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureRule" source="kind source: *v1alpha1.NodeFeatureRule"
I0504 13:53:12.044515       1 controller.go:186] "Starting Controller" logger="nfd" controller="nodefeaturerule" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureRule"
I0504 13:53:12.044535       1 leaderelection.go:250] attempting to acquire leader lease openshift-nfd/39f5e5c3.nodefeaturediscoveries.nfd.openshift.io...
I0504 13:53:12.153266       1 controller.go:220] "Starting workers" logger="nfd" controller="nodefeaturerule" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureRule" worker count=1
I0504 13:53:12.302837       1 server.go:224] "Serving metrics server" logger="nfd.controller-runtime.metrics" bindAddress="127.0.0.1:8080" secure=true
I0504 13:53:28.461923       1 leaderelection.go:260] successfully acquired lease openshift-nfd/39f5e5c3.nodefeaturediscoveries.nfd.openshift.io
I0504 13:53:28.462203       1 controller.go:178] "Starting EventSource" logger="nfd" controller="nodefeaturediscovery" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureDiscovery" source="kind source: *v1.NodeFeatureDiscovery"
I0504 13:53:28.462258       1 controller.go:178] "Starting EventSource" logger="nfd" controller="nodefeaturediscovery" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureDiscovery" source="kind source: *v1.Deployment"
I0504 13:53:28.462277       1 controller.go:178] "Starting EventSource" logger="nfd" controller="nodefeaturediscovery" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureDiscovery" source="kind source: *v1.DaemonSet"
I0504 13:53:28.462285       1 controller.go:178] "Starting EventSource" logger="nfd" controller="nodefeaturediscovery" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureDiscovery" source="kind source: *v1.ConfigMap"
I0504 13:53:28.462292       1 controller.go:178] "Starting EventSource" logger="nfd" controller="nodefeaturediscovery" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureDiscovery" source="kind source: *v1.Job"
I0504 13:53:28.462299       1 controller.go:186] "Starting Controller" logger="nfd" controller="nodefeaturediscovery" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureDiscovery"
I0504 13:53:28.572129       1 controller.go:220] "Starting workers" logger="nfd" controller="nodefeaturediscovery" controllerGroup="nfd.openshift.io" controllerKind="NodeFeatureDiscovery" worker count=1
```

8. Review the nfd-worker pod logs:

```bash
oc logs -n openshift-nfd -l app=nfd-worker --tail=1000
```

?> A successful deployment shows a similar output

```text
I0504 14:06:53.235666       1 main.go:59] "version not set! Set -ldflags \"-X github.com/openshift/node-feature-discovery/pkg/version.version=`git describe --tags --dirty --always --match 'v*'`\" during build or run."
I0504 14:06:53.236375       1 nfd-worker.go:294] "Node Feature Discovery Worker" version="undefined" nodeName="ip-10-0-29-250.us-east-2.compute.internal" namespace="openshift-nfd"
I0504 14:06:53.236657       1 nfd-worker.go:493] "configuration file parsed" path="/etc/kubernetes/node-feature-discovery/nfd-worker.conf"
I0504 14:06:53.244298       1 nfd-worker.go:528] "configuration successfully updated" configuration={"Core":{"Klog":{},"LabelWhiteList":"","NoPublish":false,"NoOwnerRefs":false,"FeatureSources":["all"],"Sources":null,"LabelSources":["all"],"SleepInterval":{"Duration":60000000000}},"Sources":{"cpu":{"cpuid":{"attributeBlacklist":["AVX10","BMI1","BMI2","CLMUL","CMOV","CX16","ERMS","F16C","HTT","LZCNT","MMX","MMXEXT","NX","POPCNT","RDRAND","RDSEED","RDTSCP","SGX","SGXLC","SSE","SSE2","SSE3","SSE4","SSE42","SSSE3","TDX_GUEST"]}},"custom":[],"fake":{"labels":{"fakefeature1":"true","fakefeature2":"true","fakefeature3":"true"},"flagFeatures":["flag_1","flag_2","flag_3"],"attributeFeatures":{"attr_1":"true","attr_2":"false","attr_3":"10"},"instanceFeatures":[{"attr_1":"true","attr_2":"false","attr_3":"10","attr_4":"foobar","name":"instance_1"},{"attr_1":"true","attr_2":"true","attr_3":"100","name":"instance_2"},{"name":"instance_3"}]},"kernel":{"KconfigFile":"","configOpts":["NO_HZ","NO_HZ_IDLE","NO_HZ_FULL","PREEMPT"]},"local":{},"pci":{"deviceClassWhitelist":["0200","03","12"],"deviceLabelFields":["vendor"]},"usb":{"deviceClassWhitelist":["0e","ef","fe","ff"],"deviceLabelFields":["class","vendor","device"]}}}
E0504 14:06:53.250001       1 memory.go:112] "failed to detect Swap nodes" err="failed to read swaps file: open /host-proc/swaps: no such file or directory"
I0504 14:06:53.274199       1 nfd-worker.go:538] "starting feature discovery..."
I0504 14:06:53.274435       1 nfd-worker.go:551] "feature discovery completed"
```

9. Finally, review the different labels in the node:

```bash
oc get nodes
oc get node <NODE_NAME>  -o jsonpath='{.metadata.labels}' | jq '.'
```

?> A successful deployment shows a similar output

```text
...
  "nvidia.com/gpu-driver-upgrade-state": "upgrade-done",
  "nvidia.com/gpu.deploy.container-toolkit": "true",
  "nvidia.com/gpu.deploy.dcgm": "true",
  "nvidia.com/gpu.deploy.dcgm-exporter": "true",
  "nvidia.com/gpu.deploy.device-plugin": "true",
  "nvidia.com/gpu.deploy.driver": "true",
  "nvidia.com/gpu.deploy.gpu-feature-discovery": "true",
  "nvidia.com/gpu.deploy.node-status-exporter": "true",
  "nvidia.com/gpu.deploy.nvsm": "",
  "nvidia.com/gpu.deploy.operator-validator": "true",
  "nvidia.com/gpu.present": "true",
...
```


## GPU Operator

Before you can use NVIDIA GPUs in OpenShift AI, you must install the NVIDIA GPU Operator. Additionally, meeting the following prerequisites is essential:

- You have logged in to your OpenShift cluster.
- You have the cluster-admin role in your OpenShift cluster.
- You have installed an NVIDIA GPU and confirmed that it is detected in your environment.

As a cluster administrator, you can install the NVIDIA GPU Operator using the OpenShift CLI (oc).

1. Create a namespace for the NVIDIA GPU Operator:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Namespace
metadata:
  name: nvidia-gpu-operator
  labels:
    name: nvidia-gpu-operator
    openshift.io/cluster-monitoring: "true"
EOF
```

2. Install the NVIDIA GPU OperatorGroup in the namespace you created in the previous step by creating the following objects:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nvidia-gpu-operator-group
  namespace: nvidia-gpu-operator
spec:
  targetNamespaces:
  - nvidia-gpu-operator
EOF
```

3. Create the following Subscription CR:

```bash
CHANNEL=$(oc get packagemanifest gpu-operator-certified -n openshift-marketplace -o jsonpath='{.status.defaultChannel}')
STARTING_CSV=$(oc get packagemanifests/gpu-operator-certified -n openshift-marketplace -ojson | jq -r '.status.channels[] | select(.name == "'$CHANNEL'") | .currentCSV')
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gpu-operator-certified
  namespace: nvidia-gpu-operator
spec:
  channel: $CHANNEL
  installPlanApproval: Automatic
  name: gpu-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
  startingCSV: $STARTING_CSV
EOF
```

4. To verify that the Operator deployment is successful, run:

```bash
oc get pods -n nvidia-gpu-operator
```

?> A successful deployment shows a Running status

```text
NAME                            READY   STATUS    RESTARTS   AGE
gpu-operator-5b47d6cd6b-lqcrh   1/1     Running   0          8m25s
```

5. Create a clusterpolicy.json file using the default config:

```bash
CHANNEL=$(oc get packagemanifest gpu-operator-certified -n openshift-marketplace -o jsonpath='{.status.defaultChannel}')
STARTING_CSV=$(oc get packagemanifests/gpu-operator-certified -n openshift-marketplace -ojson | jq -r '.status.channels[] | select(.name == "'$CHANNEL'") | .currentCSV')
oc get csv -n nvidia-gpu-operator $STARTING_CSV -o jsonpath='{.metadata.annotations.alm-examples}' | jq '.[0]' > /tmp/clusterpolicy.json
```

6. Modify the clusterpolicy.json file to specify the following configuration aspects:

```bash
vi /tmp/clusterpolicy.json
"driver": {
     "repository": "nvcr.io/nvidia",
     "image": "driver",
     "version": "570.172.08"
}
```