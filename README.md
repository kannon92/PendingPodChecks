All capitals

Example: 
- [all-capitals](./Pods/all-capitals.yaml)
  - Reproduction: Run image with invalid image name (all capitalis)
  - Pod status:
    - status: False
    - phase=
    - reason= ContainersNotReady
    - message= 'containers with unready status: [capitals]'
  - Container status:
    - state=Waiting
    - message='Failed to apply default image tag "BUSYBOX": couldn''t parse image
            reference "BUSYBOX": invalid reference format: repository name must be
            lowercase'
  - reason=InvalidImageName

- [container image does not exist](./Pods/no-exist.yaml)
  - Reproduction: Container with non existent image
  - Pod status:
    - status: False
    - phase=
    - reason= ContainersNotReady
    - message= 'containers with unready status: [capitals]'
  - Container status:
    - state=Waiting
    - message='Failed to apply default image tag "BUSYBOX": couldn''t parse image
            reference "BUSYBOX": invalid reference format: repository name must be
            lowercase'
  - reason=InvalidImageName

