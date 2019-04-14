# Nginx OSS (Open Source Software)

## Getting Started

### Install on Debian/Ubuntu 
```
apt-get update
apt-get install nginx
```
### Run via Docker
```
docker run --name some-nginx -v /some/content:/usr/share/nginx/html:ro -d nginx
```
### Verify Installation
`nginx -v`

nginx 1.15.11 is the current release. Ubuntu default repo will commonly install an older version like 1.14.0. You can install the latest by adding a repo from Nginx or run via docker which uses the latest version be default.  

### Configure Nginx
Nginx can be configured as a web server hosting static or dynamic content or perform load balancing. The same Nginx installation could be both a web server and load balancer on different ports or depending attributes such as HTTP host header. 

All configuration files for Nginx should be placed in `/etc/nginx/conf.d/`. By default, any configuration file located in this directory with a .conf extension will be loaded into Nginx on startup or by triggering a configuration reload with command `nginx -s reload`. 

Example configuration to serve static content as a web server:
```
server {
  listen 80 default_server;
  server_name www.example.com;
  location / {
    root /usr/share/nginx/html;
    # alias /usr/share/nginx/html;
    index index.html index.htm;
  }
}
```

Example configuration to load balance HTTP traffic (on port 80) to two upstream HTTP servers:
```
upstream backend {
  server 10.10.12.45:80;
  server app.example.com:80;
}
server {
  listen 80;
  location / {
    proxy_pass http://backend;
  }
}
```

Nginx OSS supports balancing methods, round robin (default), least connections, generic hash, random (my favorite), and IP hash. 

Nginx OSS supports passive health checks only. Nginx OSS cannot perform probing to determine if an upstream server is available. 

Example of passive health check configuration:
```
upstream backend {
server backend1.example.com:1234 max_fails=3 fail_timeout=3s;
server backend2.example.com:1234 max_fails=3 fail_timeout=3s;
}
```
If 3 sessions failed within 3 seconds, the upsream server will be removed from service for 3 seconds. 

### A/B testing
Nginx when acting as a load balancer can split traffic between two or more pools to facilitate A/B or canary testing. 

Example configuration of load balancing with A/B testing:
```
http {
    # ...
    # application version 1a 
    upstream version_1a {
        server 10.0.0.100:3001;
        server 10.0.0.101:3001;
    }

    # application version 1b
    upstream version_1b {
        server 10.0.0.104:6002;
        server 10.0.0.105:6002;
    }

    split_clients "{$remote_addr}" $appversion {
        95%     version_1a;
        *       version_1b;
    }

    server {
        # ...
        listen 80;
        location / {
            proxy_pass http://$appversion;
        }
    }
}
```

## More documentation on Nginx can be found on nginx.com and the O'REILLY Nginx Cookbook (free). 


