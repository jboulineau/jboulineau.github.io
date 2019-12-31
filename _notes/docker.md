(https://hortonworks.com/blog/part-5-of-data-lake-3-0-yarn-and-containerization-supporting-docker-and-beyond/)

ContainerExecutor abstraction

- Responsible for:
  - Localizing resources on nodes
  - Establishing environment required for execution (i.e. directories, etc.)
  - Managing execution lifecycle
- DockerContainerExecutor was the original incarnation
  - YARN could only use one ContainerExecutor per NodeManager, so workload types couldn't be mixed-and-matched
  - Deprecated in favor of the 'ContainerRuntime' abstraction
- LinuxContainerExecutor extended to include 'type' concept that allowed the execution of the ContainerRuntime type

https://github.com/docker/for-win/issues/522
