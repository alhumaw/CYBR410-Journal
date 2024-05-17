## Deploying Containers to Kubernetes

****

**Pods**:
- Each *pod* can have one or more containers within it.

**pod-yaml struct**
- **apiVersion**: Core kubernetes API
- **kind: Pod**: Specify what kind of resource we're creating
- **metadata**: One required field: *name*, namespace is declared here too
- **spec:** Define name and container image

Kubernetes deployments utilize the *selector* section to track and group pods

**Autoscaling** is possible
- We can declare resources to track that determine scaling in or out
	- **CPU utilization**

