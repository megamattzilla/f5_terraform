# Dockerfile for custom nginx-lb and nginx-node containers. 

## To build custom nginx containers download the Dockerfiles in this repo. This will allow you to bake your desired Nginx configuration into the container itself. 

## For a nginx container that functions as a load balancer only you could use the Dockerfile in [nginx-lb](https://github.com/megamattzilla/f5_terraform/blob/master/Dockerfile/nginx-lb/).

[nginx-lb/Dockerfile](https://github.com/megamattzilla/f5_terraform/blob/master/Dockerfile/nginx-lb/Dockerfile) contains:
```
FROM nginx

RUN rm /etc/nginx/conf.d/*

COPY app_lb.conf /etc/nginx/conf.d/
```
#### Modify app_lb.conf with your desired Nginx configuration. Then build the docker file
```
cd nginx-lb
docker build -t nginx-lb .
```
This will build an image named nginx-lb and store the image in your local docker repository. 

This container could be run with `docker run -d -p 80:80 --net=host --restart unless-stopped nginx-lb`

## Repeat the proccess to build a custom nginx container to host content [nginx-node1](https://github.com/megamattzilla/f5_terraform/tree/master/Dockerfile/nginx-node1).

[nginx-node1/Dockerfile](https://github.com/megamattzilla/f5_terraform/blob/master/Dockerfile/nginx-node1/Dockerfile) contains:
```
FROM nginx

RUN rm /etc/nginx/conf.d/*

RUN mkdir /usr/share/nginx/html/app1

COPY index.nginx-debian.html /usr/share/nginx/html/app1/

COPY NGINX-Cookbook-2019.pdf /usr/share/nginx/html/app1/

COPY node1.conf /etc/nginx/conf.d/
```
#### Modify node1.conf with your desired Nginx configuration and change the HTML content in index.nginx-debian.html if desired. Then build the docker file
```
cd nginx-node1
docker build -t nginx-node1 .
```
This will build an image named nginx-node1 and store the image in your local docker repository. 

This container could be run with `docker run -d -p 9001:9001 --name=nginx-node1 --restart unless-stopped nginx-node1`

### These container could optionally be uploaded to Docker Hub using a free Docker account. 




