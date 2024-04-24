
### Starting a Kubernetes stack

- **Build the container**
	- `cd ~/path/to/dockerfile`
	- `docker build -t web .`
- **Create the registry**
- `microk8s.enable registry dns`
- **Tag/push the container**
	- `docker tag web localhost:32000/web:k8s`
	- `docker push localhost:32000/web:k8s`
- **Modify test.yaml to your docker tag name**
	- `image: localhost:32000/web:k8s`
- **Spin up your kubernetes stack**
	- `microk8s.kubectl apply -f test.yaml`  
	- `microk8s.kubectl apply -f test-svc.yaml`
- **Spin down your kubernetes stack**
	- `microk8s.kubectl delete -f test-svc.yaml`
	- `microk8s.kubectl delete -f test.yaml`

