# Conditions and Statues for Pending Pods

## Motivation

As a batch system developer, we find that higher level controllers have to write code to handle Pending Pod cases.  Since the pods are not considered failed, the higher level controller needs to transition this state to failed in order to enforce retries.  For example, the job controller will run these pods and the state of the Pod will be in Pending.  The job controller does not transition this state to failed even if the pod will never.

In the Armada project, we implement logic that determines retry ability based on events and statues.  Events are not standard so we expect issues when updating.  

It would be ideal to have a standard condition/status for these common configuration errors.

It would also be worth exploring if certain states can transition to failed as part of kubelet path.

## Findings from Examples

- Invalid Image Name
  - Should always fail
  - [image-manager](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/images/image_manager.go#L107)
  - State=Waiting with Reason providing information
- ImagePullBackOff
  - invalid image, pull secrets, overloaded registry
  - This one may be difficult to predict if it will always fail
- Invalid Config Map
  - Reason=CreateContainerConfigError
  - [CreateContainerConfigError](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/kuberuntime/kuberuntime_container.go#L173)
  - Status shows the Waiting reason
  - Invalid key references falls under this also
- Invalid Mounting Volumes from secrets or config-maps
  - Happens if you have valid volume names but missing config-map or secret
  - There is no container statues for this one
  - Status is ContainerCreating
  - Event says "MountVolume.SetUp failed for volume ..."
  -[MountVolume.SetUp](https://github.com/kubernetes/kubernetes/blob/master/pkg/volume/util/operationexecutor/operation_generator.go#L530)
- Unable to Schedule (missing Volume)
  - Condition is unable to schedule
  - Fails in scheduling stage.

## Examples
In this section, we will demonstate how to reproduce certain pending conditions and write out the conditions, status and events if necessary.
- [all-capitals](./Pods/All-Capitals/all-capitals.yaml)
  - Reproduction: Run image with invalid image name (all capitals)
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - state=Waiting
      - message='Failed to apply default image tag "BUSYBOX": couldn''t parse image
            reference "BUSYBOX": invalid reference format: repository name must be
            lowercase'
      - reason=InvalidImageName
  - Conditions:
    - Type=Status
    - Initialized=True
    - Ready=False
    - ContainersReady=False
    - PodScheduled=True
  - Get Table (`kubectl get pods`)
    - Ready=0/1
    - Status=InvalidImageName
    - Restarts=0
- [no-exist](./Pods/NoExist/no-exist.yaml)
  - Reproduction: Container with non existent image
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - state=Waiting
      - message=Back-off pulling image
      - reason=ImagePullBackOff
  - Conditions:
    - Type=Status
    - Initialized=True
    - Ready=False
    - ContainersReady=False
    - PodScheduled=True
  - Get Table
    - Ready=0/1
    - Status=ImagePullBackOff
- [private-docker-no-crds](./Pods/Container-Private/container-private.yaml)
  - Reproduction: Container with private image
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - state=Waiting
      - message=Back-off pulling image
      - reason=ImagePullBackOff
  - Conditions:
    - Type=Status
    - Initialized=True
    - Ready=False
    - ContainersReady=False
    - PodScheduled=True
  - Get Table
    - Ready=0/1
    - Status=ImagePullBackOff
- [config-map-missing](./Pods/Config-Map-Missing/config-map-missing.yaml)
  - Reproduction: Container with missing config map
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - state=Waiting
      - message=configmap "%s" not found
      - reason=CreateContainerConfigError
  - Conditions:
    - Type=Status
    - Initialized=True
    - Ready=False
    - ContainersReady=False
    - PodScheduled=True
  - Get Table
    - Ready=0/1
    - Status=CreateContainerConfigError
- [config-map-key-missing](./Pods/Config-Map-Found-Missing-Key/config-map-found-missing-key.yaml)
  - Reproduction: Container with existing config map but invalid key reference
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - state=Waiting
      - message=couldn't find key special.how in ConfigMap default/special-config
      - reason=CreateContainerConfigError
  - Conditions:
    - Type=Status
    - Initialized=True
    - Ready=False
    - ContainersReady=False
    - PodScheduled=True
  - Get Table
    - Ready=0/1
    - Status=CreateContainerConfigError

- [volume-missing](./Pods/Volume-Missing/volume-missing.yaml)
  - Reproduction: Container with missing volume claim
  - This has no container status and it actually fails in the scheduling step.
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - Does not Exist for this case
  - Conditions:
    - PodScheduled=False
    - message='0/1 nodes are available: 1 persistentvolumeclaim "example-pvc" not found.
      preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.'
    - reason: Unschedulable
    - status: "False
    - type: PodScheduled
  - Get Table
    - Ready=0/1
    - Status=Pending
- [volume-missing-secret](./Pods/Volume-Secret-Mount/volume-secret-mount.yaml)
  - Reproduction: Container with missing secret for volume
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - state=Waiting
      - Reason=ContainerCreating
  - Conditions:
    - Type=Status
    - Initialized=True
    - Ready=False
    - ContainersReady=False
    - PodScheduled=True
  - Get Table
    - Ready=0/1
    - Status=ContainerCreating
  - Events show a message for MountVolume.SetUp
  - "MountVolume.SetUp failed for volume "secret-volume" : secret "test-secret" not found"
  - This does not show up as reason or error.
- [config-map-volume-missing](./Pods/Config-Map-Mount-Volume/config-map-mount-volume.yaml)
  - Reproduction: Container with missing secret for volume
  - Pod status:
    - status: Pending
  - ContainerStatus:
    - state=Waiting
      - Reason=ContainerCreating
  - Conditions:
    - Type=Status
    - Initialized=True
    - Ready=False
    - ContainersReady=False
    - PodScheduled=True
  - Get Table
    - Ready=0/1
    - Status=ContainerCreating
  - Events show a message for MountVolume.SetUp
  - MountVolume.SetUp failed for volume "clusters-config-volume" : configmap "clusters-config-file" not found
  - This does not show up as reason or error.

  

