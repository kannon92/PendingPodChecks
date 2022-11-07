# Conditions and Statues for Pending Pods

## Motivation

As a batch system developer, we find that higher level controllers have to write code to handle Pending Pod cases.  Since the pods are not considered failed, the higher level controller needs to transition this state to failed in order to enforce retries.  For example, the job controller will run these pods and the state of the Pod will be in Pending.  The job controller does not transition this state to failed even if the pod will never.

In the Armada project, we implement logic that determines retry ability based on events and statues.  Events are not standard so we expect issues when updating.  

It would be ideal to have a standard condition/status for these common configuration errors 

Example: 
- [all-capitals](./Pods/All-Capitals/all-capitals.yaml)
  - Reproduction: Run image with invalid image name (all capitalis)
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
  

