# Secure Web Server Implementation
###### Eastern Washington University
****
## Kubernetes & Docker Guide
- Update
	- `sudo apt update -y`
- Install docker
	- `sudo apt install docker.io -y`
- Add user to docker group
	- `sudo usermod -a -G docker $USER`
- Install microk8s
	- `sudo snap install microk8s --classic`
- Install kubectl
	- `sudo snap install kubectl --classic`
- Install git
	- `sudo apt install git -y`
- Add user to microk8s group
	- `sudo usermod -a -G microk8s $USER`
	- `newgrp microk8s`
- Enable the microk8s registry
	- `microk8s.enable registry`
- Clone my repository
	- `git clone https://github.com/alhumaw/Big-Deployment`
- Build the container
	- `cd Big-Deployment/src`
	- `docker build -t localhost:32000/flask:k8s .`
- Push the container
	- `docker push localhost:32000/flask:k8s`
- Deploy kubernetes
	- `cd ~/Big-Deployment/kube/demo`
	- `microk8s.kubectl apply -f flask-deploy.yaml`
	- `microk8s.kubectl apply -f balance-svc.yaml`

## NGINX Setup Guide
- `sudo apt update -y`
- `sudo apt install nginx -y`
- Our **NGINX prox**y will work to serve content to users by grabbing it from our load balancer. We just need to clone a repository I created and place the file into: `/etc/nginx/conf.d`
	- `sudo cp ~/Big-Deployment/proxy.conf /etc/nginx/conf.d/`
	- We have to change the server IP to match our load balancer
		- `microk8s.kubectl get services | grep flask | awk '{print $3}'`
		- Replace the old IP address with the output of the above command in proxy.conf
- **To set up the WAF**, we have to follow a particular series of steps:
	- Update
		- `sudo apt-get update -y`

	- Install build dependencies
		- `sudo apt-get install bison build-essential ca-certificates curl dh-autoreconf doxygen flex gawk git iputils-ping libcurl4-gnutls-dev libexpat1-dev libgeoip-dev liblmdb-dev libpcre3-dev libssl-dev libtool libxml2 libxml2-dev libyajl-dev locales lua5.3-dev pkg-config wget zlib1g-dev libgd-dev m4 automake g++`
	- Clone the modsec repo
		- `cd /opt && sudo git clone https://github.com/SpiderLabs/ModSecurity`
		- `cd ModSecurity`
	- Initialize and update submodule
		- `sudo git submodule init; sudo git submodule update`
	- Run the build script
		- `sudo ./build.sh`
	- Run the configure file
		- `sudo ./configure`
	- Run make 
		- `sudo make`
	- Install
		- `sudo make install`
	- Clone the nginx-connector
		- `cd /opt && sudo git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git`
	- Download a copy of our nginx version
		- `cd /opt && sudo wget http://nginx.org/download/nginx-1.24.0.tar.gz`
	- Extract the tarfile
		- `sudo tar -xvzmf nginx-1.24.0.tar.gz`
		- Change Directory
			- `cd nginx-1.24.0.tar.gz`
		- Grab nginx version
			- `nginx -V`
		- Copy the configure arguments starting with and including `--with-cc-opt=`
		- Before `--with-cc-opt=`, add `--add-dynamic-module=../ModSecurity-nginx`
		- Build the modules:
			- `sudo make modules`
		- Create a directory for modules in the NGINX directory
			- `sudo mkdir /etc/nginx/modules`
		- Copy the ModSec module into the NGINX configuration folder:
			- `sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules`
		- Prepend the module into our NGINX configuration file (/etc/nginx/nginx.conf)
			- `load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;`
	- From this point, we will add a rule set for our WAF to utilize when examining HTTP content received. 
		- Open /etc/nginx/nginx.conf and add the following line below the `include` statement
			- `load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;`
		- Delete the current ModSec rulelist
			- `sudo rm -rf /usr/share/modsecurity-crs`
		- Clone the OWASP-CRS GitHub repository into the /usr/share/modsecurity-crs directory:
			- `sudo git clone https://github.com/coreruleset/coreruleset /usr/local/modsecurity-crs`
		- Rename
			- `sudo mv /usr/local/modsecurity-crs/crs-setup.conf.example /usr/local/modsecurity-crs/crs-setup.conf`
		- Rename
			- `sudo mv /usr/local/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example /usr/local/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf`
	- Now we are going to configure modsec
		- make directory
			- `sudo mkdir -p /etc/nginx/modsec`
		- Unicode mapping stuff
			- `sudo cp /opt/ModSecurity/unicode.mapping /etc/nginx/modsec `
			- `sudo cp /opt/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
		- Open the `/etc/nginx/modsecurity.conf` config file in a text editor and change `SecEngineRule` to `On`.
	- Create a new configuration file called `main.conf` under the `/etc/nginx/modsec` directory:
		- `sudo touch /etc/nginx/modsec/main.conf`
	- Open the configuration file in a text editor and add these lines of text:
		- `Include /etc/nginx/modsec/modsecurity.conf 
		- `Include /usr/local/modsecurity-crs/crs-setup.conf` 
		- `Include /usr/local/modsecurity-crs/rules/*.conf`
- `sudo systemctl restart nginx`
- The service is now acting as a proxy that is managed by a Web Application Firewall
