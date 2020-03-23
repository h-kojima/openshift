FROM docker.io/library/centos:centos6

RUN yum -y install epel-release; \
    yum -y install nginx
RUN mkdir -p /var/www; \
    echo 'Hello, World!' > /var/www/index.html
ADD nginx.conf /
RUN chmod ugo+r /nginx.conf; \
    chmod -R ugo+r /var/www

USER 10001
EXPOSE 8080
CMD ["/usr/sbin/nginx", "-c", "/nginx.conf", "-g", "daemon off;"]
