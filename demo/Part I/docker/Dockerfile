FROM ubuntu:20.04

RUN apt update
RUN apt -y install nginx

COPY ./test.html /var/www/html/index.html

CMD ["/bin/bash", "-c", "nginx -g \"daemon off;\""]