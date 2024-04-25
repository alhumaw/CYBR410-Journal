
### Part 1 simple container:
```
First off you will need a container that has a web service you want to host. I used my container from earlier in the week. It’s an ubuntu container that is running a python web server on port 8000. You can do whatever you want, but it must be a web service of some kind that has a port open with which to access it.
```

- To get started with this lab, I just utilized the web server container we created this last week.
- `docker images`
- ![[Pasted image 20240423103223.png]]
- The container is running a python web server, hosting it's contents on port 8000. 

### Part 2 k8s:
- To spin up a kubernetes stack, we need to host a server that will hold the containers that we create:
	- `microk8s.enable registry dns`
- Once that command has finished executing, we can push the docker image we created into the registry:
	- `docker tag web localhost:32000/web:k8s`
	- `docker push localhost:32000/web:k8s`
	- The `localhost:32000` section just means that we will host our web server locally on that specific port.
	- `docker tag` is used to create a tagline for our container that we can refer to as a specific version, or image, of our container.
	- `docker push` pushes our image to a registry, like the one we created above.
- These three commands ensure secure version control and kubernetes interactivity.

### Part 3 k8s config files:
- We were given two file to modify: `test.yaml` and `test-svc.yaml`
- These two config files allow us to start the pod and service to spin up our web server on kubernetes
- All we have to do is replace a single line in the config file to match our tagged container:
	- `image: localhost:32000/web:k8s`
	- ![[testyaml.png]]
- With this out of the way all we have to do is use two commands to spin things up:
	- `microk8s.kubectl apply -f test.yaml`
	- `microk8s.kubectl apply -f test-svc.yaml`
- If everything was executed correctly, we should be able to see our pods using these two commands:
	- `microk8s.kubectl get pods --all-namespaces`
	- `microk8s.kubectl get services --all-namespaces`
	- ![[Pasted image 20240423112947.png]]

### Writeups
- **In the test-svc.yaml file, what does type: LoadBalancer mean?**
	- The `LoadBalancer` type indicates that the service should be load-balanced.
	- This essentially means that our endpoints can distribute client load evenly across all our pods such that no specific pod is overloaded.
- **How many instances of the container are running?**
	- There are three instances of our container running. This can be increased or decreased by modifying the `replicas` tag in `test.yaml`
- **What happens if one of these containers is destroyed?**
	- If one of the containers is destroyed, kubernetes orchestration is monitoring our pods. The service will work to consistently run the amount of containers we request so that container will come back up.
	- The load balancer will redirect traffic to the other pods.
	- The containers health will be displayed in our pod list, indicating a specific issue.

![[Pasted image 20240423115613.png]]

