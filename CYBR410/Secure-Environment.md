# Secure Web Server Implementation
###### Eastern Washington University
****
## Introduction
- The implementation for the environment will be hosted on an Ubuntu server. We will host our website using a kubernetes cluster and load balancer to ensure high availability. Our content will be served through an NGINX proxy that also hosts a Web Application Firewall (WAF) through ModSecurity to prevent common OWASP exploitation strategies. We also will implement a rate-limiting feature to prevent DoS/Scraping/Spider attacks. This implementation should be more than enough to keep our website up an running without issues.
- Outside of our web security, we will purposefully open ports for attackers to attempt to connect on. Utilizing sysdig, we will catch the IP addresses of the attackers and block their IP address on our server using IPtables.
- Our Apache frontend will be hosted by kubernetes, it can legitimately be anything we just need it to serve interesting content. 

## NGINX Setup Guide
- `sudo apt update -y`
- `sudo apt install nginx -y`
- Our **NGINX prox**y will work to serve content to users by grabbing it from our load balancer. We just need to clone a repository I created and place the file into: `/etc/nginx/conf.d`
	- `git clone https://github.com/alhumaw/proxy-nginx`
	- `sudo mv proxy-nginx/proxy.conf /etc/nginx/conf.d/`
	- We will have to change the server IP to match our load balancer once it's up and running.
- **To set up the WAF**, we have to follow a particular series of steps:
	- The series of steps can be found: https://www.linode.com/docs/guides/securing-nginx-with-modsecurity/
	- Once you reach the `nginx -V` part, there is an issue with the geoIP module that I found a workaround to:
		- `sudo apt install -y libmaxminddb0 libmaxminddb-dev mmdb-bin build-essential libpcre3-dev zlib1g-dev libssl-dev libxml2-dev libxslt-dev libgd-dev`
		- `cd nginx-1.18.0`
		- `wget https://github.com/leev/ngx_http_geoip2_module/archive/refs/tags/3.3.tar.gz`
		- `tar xvfz 3.3.tar.gz`
		- `nginx -V`
		- Copy the configure arguments
		- Run the command
		- `sudo ./configure --add-dynamic-module=./ngx_http_geoip2_module-3.3 --add-dynamic-module=../ModSecurity-nginx --with-cc-opt='-g -O2 -ffile-prefix-map=/build/nginx-zctdR4/nginx-1.18.0=. -flto=auto -ffat-lto-objects -flto=auto -ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security -fPIC -Wdate-time -D_FORTIFY_SOURCE=2' --with-ld-opt='-Wl,-Bsymbolic-functions -flto=auto -ffat-lto-objects -flto=auto -Wl,-z,relro -Wl,-z,now -fPIC' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --modules-path=/usr/lib/nginx/modules --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-compat --with-debug --with-pcre-jit --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_v2_module --with-http_dav_module --with-http_slice_module --with-threads --with-http_addition_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_sub_module`
		- Build the modules:
			- `sudo make modules`
		- Create a directory for modules in the NGINX directory
			- `sudo mkdir /etc/nginx/modules`
		- Copy the ModSec module into the NGINX configuration folder:
			- `sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules`
		- Prepend the module into our NGINX configuration file (/etc/nginx/nginx.conf)
			- `load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;`
	- From this point, we will add a rule set for our WAF to utilize when examining HTTP content received. 
		- Follow https://www.linode.com/docs/guides/securing-nginx-with-modsecurity/#setting-up-owasp-crs
	- Once you get to: https://www.linode.com/docs/guides/securing-nginx-with-modsecurity/#configuring-modsecurity, ignore this step:
		- ` sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf`
	- Open the modsecurity config file in a text editor and change `SecEngineRule` to `On`.
	- Create a new configuration file called `main.conf` under the `/etc/nginx/modsec` directory:
		- `sudo touch /etc/nginx/modsec/main.conf`
	- Open the configuration file in a text editor and add these lines of text:
		- `Include /etc/nginx/modsec/modsecurity.conf 
		- `Include /usr/local/modsecurity-crs/crs-setup.conf` 
		- `Include /usr/local/modsecurity-crs/rules/*.conf`
- `sudo systemctl restart nginx`
- The service is now acting as a proxy that is managed by a Web Application Firewall

## Logging Implementation
- There is a filter we will have to come up with that will identify incoming connections not on port 80. We will grab the IP addresses of those results and add them to a blocklist via IPTables
- This will effectively act as a honeypot for anyone attempting to access anything other than our website.



![[Screenshot 2024-05-15 at 8.42.38 PM.png]]