FROM nginx

RUN rm /etc/nginx/conf.d/*

RUN mkdir /usr/share/nginx/html/app1

COPY index.nginx-debian.html /usr/share/nginx/html/app1/

COPY NGINX-Cookbook-2019.pdf /usr/share/nginx/html/app1/

COPY node1.conf /etc/nginx/conf.d/
