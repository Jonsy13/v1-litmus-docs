---
id: version-1.13.8-pod-memory-hog
title: Pod Memory Hog Details
sidebar_label: Pod Memory Hog
original_id: pod-memory-hog
---
------

## Experiment Metadata

<table>
  <tr>
    <th> Type </th>
    <th> Description </th>
    <th> Tested K8s Platform </th>
  </tr>
  <tr>
     <td> Generic </td>
    <td> Consume memory resources on the application container</td>
    <td> GKE, Packet(Kubeadm), Minikube, EKS, AKS  </td>
  </tr>
</table>

## Prerequisites

- Ensure that Kubernetes Version > 1.16
- Ensure that the Litmus Chaos Operator is running by executing `kubectl get pods` in operator namespace (typically, `litmus`). If not, install from [here](https://docs.litmuschaos.io/docs/getstarted/#install-litmus)
- Ensure that the `pod-memory-hog` experiment resource is available in the cluster by executing `kubectl get chaosexperiments` in the desired namespace. If not, install from [here](https://hub.litmuschaos.io/api/chaos/1.13.8?file=charts/generic/pod-memory-hog/experiment.yaml)
- Cluster must run docker container runtime

## Entry Criteria

- Application pods are healthy on the respective nodes before chaos injection

## Exit Criteria

- Application pods are healthy on the respective nodes post chaos injection

## Details

- This experiment consumes the Memory resources on the application container on specified memory in megabytes.
- It simulates conditions where app pods experience Memory spikes either due to expected/undesired processes thereby testing how the overall application stack behaves when this occurs.


## Integrations

- Pod Memory can be effected using the chaos library: `litmus`

## Steps to Execute the Chaos Experiment

- This Chaos Experiment can be triggered by creating a ChaosEngine resource on the cluster. To understand the values to provide in a ChaosEngine specification, refer [Getting Started](getstarted.md/#prepare-chaosengine)

- Follow the steps in the sections below to create the chaosServiceAccount, prepare the ChaosEngine & execute the experiment.

### Prepare chaosServiceAccount

Use this sample RBAC manifest to create a chaosServiceAccount in the desired (app) namespace. This example consists of the minimum necessary role permissions to execute the experiment.

#### Sample Rbac Manifest

[embedmd]:# (https://raw.githubusercontent.com/litmuschaos/chaos-charts/v1.13.x/charts/generic/pod-memory-hog/rbac.yaml yaml)
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-memory-hog-sa
  namespace: default
  labels:
    name: pod-memory-hog-sa
    app.kubernetes.io/part-of: litmus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-memory-hog-sa
  namespace: default
  labels:
    name: pod-memory-hog-sa
    app.kubernetes.io/part-of: litmus
rules:
- apiGroups: [""]
  resources: ["pods","events"]
  verbs: ["create","list","get","patch","update","delete","deletecollection"]
- apiGroups: [""]
  resources: ["pods/exec","pods/log","replicationcontrollers"]
  verbs: ["create","list","get"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["create","list","get","delete","deletecollection"]
- apiGroups: ["apps"]
  resources: ["deployments","statefulsets","daemonsets","replicasets"]
  verbs: ["list","get"]
- apiGroups: ["apps.openshift.io"]
  resources: ["deploymentconfigs"]
  verbs: ["list","get"]
- apiGroups: ["argoproj.io"]
  resources: ["rollouts"]
  verbs: ["list","get"]
- apiGroups: ["litmuschaos.io"]
  resources: ["chaosengines","chaosexperiments","chaosresults"]
  verbs: ["create","list","get","patch","update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-memory-hog-sa
  namespace: default
  labels:
    name: pod-memory-hog-sa
    app.kubernetes.io/part-of: litmus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-memory-hog-sa
subjects:
- kind: ServiceAccount
  name: pod-memory-hog-sa
  namespace: default
```

***Note:*** In case of restricted systems/setup, create a PodSecurityPolicy(psp) with the required permissions. The `chaosServiceAccount` can subscribe to work around the respective limitations. An example of a standard psp that can be used for litmus chaos experiments can be found [here](https://docs.litmuschaos.io/docs/next/litmus-psp/).

### Prepare ChaosEngine

- Provide the application info in `spec.appinfo`
- Override the experiment tunables if desired in `experiments.spec.components.env`
- To understand the values to provided in a ChaosEngine specification, refer [ChaosEngine Concepts](chaosengine-concepts.md)

#### Supported Experiment Tunables

<table>
  <tr>
    <th>  Variables </th>
    <th>  Description </th>
    <th> Type  </th>
    <th> Notes </th>
  </tr>
  <tr>
    <td> MEMORY_CONSUMPTION </td>
    <td>  The amount of memory used of hogging a Kubernetes pod (megabytes)</td>
    <td> Optional  </td>
    <td> Defaults to 500MB (Up to 2000MB)</td>
  </tr>
  <tr>
    <td> NUMBER_OF_WORKERS </td>
    <td> The number of workers used to run the stress process  </td>
    <td> Optional </td>
    <td> Defaults to 1 </td>
  </tr>  
  <tr>
    <td> TOTAL_CHAOS_DURATION </td>
    <td> The time duration for chaos insertion (seconds)  </td>
    <td> Optional </td>
    <td> Defaults to 60s </td>
  </tr>
    <td> LIB  </td>
    <td> The chaos lib used to inject the chaos. Available libs are <code>litmus</code> and <code>pumba</code> </td>
    <td> Optional </td>
    <td> Defaults to <code>litmus</code> </td>
  </tr>
   <tr>
    <td> LIB_IMAGE  </td>
    <td> Image used to run the helper pod.</td>
    <td> Optional  </td>
    <td> Defaults to <code>litmuschaos/go-runner:1.13.8<code> </td>
  </tr>
   <tr>
    <td> STRESS_IMAGE  </td>
    <td> Container run on the node at runtime by the pumba lib to inject stressors. Only used in LIB <code>pumba</code></td>
    <td> Optional  </td>
    <td> Default to <code>alexeiled/stress-ng:latest-ubuntu</code> </td>
  </tr>
  <tr>
    <td> TARGET_PODS </td>
    <td> Comma separated list of application pod name subjected to pod memory hog chaos</td>
    <td> Optional </td>
    <td> If not provided, it will select target pods randomly based on provided appLabels</td>
  </tr>
  <tr>
    <td> TARGET_CONTAINER </td>
    <td> Name of the target container under chaos.</td>
    <td> Optional </td>
    <td> If not provided, it will select the first container of the target pod</td>
  </tr>   
  <tr>
    <td> CONTAINER_RUNTIME  </td>
    <td> container runtime interface for the cluster</td>
    <td> Optional </td>
    <td> Defaults to docker, supported values: docker, containerd and crio for litmus and only docker for pumba LIB </td>
  </tr>
  <tr>
    <td> SOCKET_PATH </td>
    <td> Path of the containerd/crio/docker socket file </td>
    <td> Optional  </td>
    <td> Defaults to `/var/run/docker.sock` </td>
  </tr>          
  <tr>
    <td> PODS_AFFECTED_PERC </td>
    <td> The Percentage of total pods to target  </td>
    <td> Optional </td>
    <td> Defaults to 0 (corresponds to 1 replica), provide numeric value only </td>
  </tr>  
  <tr>
    <td> RAMP_TIME </td>
    <td> Period to wait before and after injection of chaos in sec </td>
    <td> Optional  </td>
    <td> </td>
  </tr>
  <tr>
    <td> SEQUENCE </td>
    <td> It defines sequence of chaos execution for multiple target pods </td>
    <td> Optional </td>
    <td> Default value: parallel. Supported: serial, parallel</td>
  </tr>
  <tr>
    <td> INSTANCE_ID </td>
    <td> A user-defined string that holds metadata/info about current run/instance of chaos. Ex: 04-05-2020-9-00. This string is appended as suffix in the chaosresult CR name.</td>
    <td> Optional </td>
    <td> Ensure that the overall length of the chaosresult CR is still < 64 characters </td>
  </tr>

</table>

#### Sample ChaosEngine Manifest

[embedmd]:# (https://raw.githubusercontent.com/litmuschaos/chaos-charts/v1.13.x/charts/generic/pod-memory-hog/engine.yaml yaml)
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
  namespace: default
spec:
  # It can be active/stop
  engineState: 'active'
  appinfo:
    appns: 'default'
    applabel: 'app=nginx'
    appkind: 'deployment'
  chaosServiceAccount: pod-memory-hog-sa
  experiments:
    - name: pod-memory-hog
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '60' # in seconds

            # Enter the amount of memory in megabytes to be consumed by the application pod
            - name: MEMORY_CONSUMPTION
              value: '500'

             ## percentage of total pods to target
            - name: PODS_AFFECTED_PERC
              value: ''   

            ## provide the cluster runtime
            - name: CONTAINER_RUNTIME
              value: 'docker'   

            # provide the socket file path
            - name: SOCKET_PATH
              value: '/var/run/docker.sock'              
```

### Create the ChaosEngine Resource

- Create the ChaosEngine manifest prepared in the previous step to trigger the Chaos.

  `kubectl apply -f chaosengine.yml`

- If the chaos experiment is not executed, refer to the [troubleshooting](https://docs.litmuschaos.io/docs/faq-troubleshooting/)
  section to identify the root cause and fix the issues.

### Watch Chaos progress

- Set up a watch on the applications interacting/dependent on the affected pods and verify whether they are running

  `watch kubectl get pods -n <application-namespace>`

### Abort/Restart the Chaos Experiment

- To stop the pod-memory-hog experiment immediately, either delete the ChaosEngine resource or execute the following command: 

  `kubectl patch chaosengine <chaosengine-name> -n <namespace> --type merge --patch '{"spec":{"engineState":"stop"}}'` 

- To restart the experiment, either re-apply the ChaosEngine YAML or execute the following command: 

  `kubectl patch chaosengine <chaosengine-name> -n <namespace> --type merge --patch '{"spec":{"engineState":"active"}}'` 

### Check Chaos Experiment Result

- Check whether the application stack is resilient to Memory spikes on the app replica, once the experiment (job) is completed. The ChaosResult resource name is derived like this: `<ChaosEngine-Name>-<ChaosExperiment-Name>`.

  `kubectl describe chaosresult nginx-chaos-pod-memory-hog -n <application-namespace>`

## Pod Memory Hog Experiment Demo

- A sample recording of this experiment execution is provided [here](https://www.youtube.com/watch?v=HuAXg8W5Tzo)
