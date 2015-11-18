Notes on how to set up jenkins+docker environment


http://engineering.riotgames.com/news/putting-jenkins-docker-container


#Run docker:
docker-machine start default

#Checkout env:
docker-machine env default

#Setup env:
eval "$(docker-machine env default)"

#Check ip
docker-machine ip

#Verify that docker is working
docker ps

#Pull jenkins from public repo, and run with default settings at port 8080

docker pull jenkins
docker run -p 8080:8080 --name=jenkins-master jenkins

#Remember that this is not running at localhost, but virtual ip you got earlier, for example
http://192.168.99.100:8080/

#Daemonizing docker:
docker rm jenkins-master
docker run -p 8080:8080 --name=jenkins-master -d jenkins

#More memory for Docker:
docker stop jenkins-master
docker rm jenkins-master
docker run -p 8080:8080 --name=jenkins-master -d --env JAVA_OPTS="-Xmx8192m" jenkins

#Larger connection pool for Docker:
docker stop jenkins-master
docker rm jenkins-master
docker run -p 8080:8080 --name=jenkins-master -d --env JAVA_OPTS="-Xmx8192m" --env JENKINS_OPTS="--handlerCountStartup=100 --handlerCountMax=300" jenkins

#Build own dockerfile

#Create folder, and file named Dockerfile, with this contents:
FROM jenkins:1.609.1
MAINTAINER artosa

#Build own docker image
docker build -t myjenkins .

#Run own docker:
docker run -p 8080:8080 --name=jenkins-master -d --env JAVA_OPTS="-Xmx8192m" --env JENKINS_OPTS="--handlerCountStartup=100 --handlerCountMax=300" myjenkins

#add this to Dockerfile:
ENV JAVA_OPTS="-Xmx8192m"
ENV JENKINS_OPTS="--handlerCountStartup=100 --handlerCountMax=300 --logfile=/var/log/jenkins/jenkins.log"

#Build Docker
docker build -t myjenkins .

#Stop and move configs to Dockerfile:
docker stop jenkins-master
docker rm jenkins-master

#Run again with these modifications:
docker run -p 8080:8080 --name=jenkins-master -d myjenkins

#Check that options are set right:
docker exec jenkins-master ps -ef | grep java


#Edit Dockerfile to create a folder for logs
USER root
RUN mkdir /var/log/jenkins
RUN chown -R  jenkins:jenkins /var/log/jenkins
USER jenkins

#Change JENKINS_OPTS line to:
ENV JENKINS_OPTS="--handlerCountStartup=100 --handlerCountMax=300 --logfile=/var/log/jenkins/jenkins.log"

#Build, stop, test again
docker build -t myjenkins .
docker stop jenkins-master
docker rm jenkins-master
docker run -p 8080:8080 --name=jenkins-master -d myjenkins

#Check the logfile
docker exec jenkins-master tail -f /var/log/jenkins/jenkins.log


#Now it's possible to create a data volume, as a separate container, like this:

#New Dockerfile:
FROM debian:jessie
MAINTAINER artosa
RUN useradd -d "/var/jenkins_home" -u 1000 -m -s /bin/bash jenkins
RUN mkdir -p /var/log/jenkins
RUN chown -R jenkins:jenkins /var/log/jenkins
VOLUME ["/var/log/jenkins"]
USER jenkins
CMD ["echo", "Data container for Jenkins"]

#Build and run:
docker build -t myjenkinsdata jenkins-data/.
docker run --name=jenkins-data myjenkinsdata


#To use the new data container, we can do:
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d myjenkins

#Port 50000 is for JNLP build slaves, not used yet

#Verify that all still works
docker exec jenkins-master tail -f /var/log/jenkins/jenkins.log

#Prove that container is in use and works, you should see log getting appended instead of overwritten

docker stop jenkins-master
docker rm jenkins-master
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d myjenkins
docker exec jenkins-master cat /var/log/jenkins/jenkins.log

#Here's how to copy stuff from docker containers:

docker cp jenkins-data:/var/log/jenkins/jenkins.log jenkins.log

#Move Jenkins data dir to external volume by changing jenkins-data Dockerfile VOLUME line like this:
VOLUME ["/var/log/jenkins", "/var/jenkins_home"]

#Rebuild, cleanup
docker rm jenkins-data
docker build -t myjenkinsdata jenkins-data/.
docker run --name=jenkins-data myjenkinsdata

#Make Jenkins move its war file from jenkins_home to /var/cache/jenkins/war by editing jenkins-master dockerfile and changing JENKINS_OPTS:

ENV JENKINS_OPTS="--handlerCountStartup=100 --handlerCountMax=300 --logfile=/var/log/jenkins/jenkins.log --webroot=/var/cache/jenkins/war"


#Also, change Dockerfile to create this folder and assign permissions to Jenkins:
USER root
RUN mkdir /var/log/jenkins
RUN mkdir /var/cache/jenkins
RUN chown -R  jenkins:jenkins /var/log/jenkins
RUN chown -R jenkins:jenkins /var/cache/jenkins
USER jenkins


#Rebuild jenins master, restart it
#Note how -v switch will remove also data volume when you remove the master
docker stop jenkins-master
docker rm -v jenkins-master
docker build -t myjenkins jenkins-master/.
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d myjenkins

#Verify that jenkins war is exploded in the cache
docker exec jenkins-master ls /var/cache/jenkins/war

#Test: Log into jenkins, create an empty freestyle jos, then stop, remove, and restart jenkins-master:
docker stop jenkins-master
docker rm jenkins-master
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d myjenkins

#Tah-dah! Job's still there!







#Full dockerfiles for stage 2 here, jenkins-master:
#Cloudbees Jenkins image

FROM jenkins:1.609.1
MAINTAINER artosa

USER root
RUN mkdir /var/log/jenkins
RUN mkdir /var/cache/jenkins
RUN chown -R  jenkins:jenkins /var/log/jenkins
RUN chown -R jenkins:jenkins /var/cache/jenkins
USER jenkins

ENV JAVA_OPTS="-Xmx8192m"
ENV JENKINS_OPTS="--handlerCountStartup=100 --handlerCountMax=300 --logfile=/var/log/jenkins/jenkins.log --webroot=/var/cache/jenkins/war"





#Jenkins-data:
# NOTE: I use the base Debian image because it matches the same base image 
# the Cloudbees Jenkins image uses. Because we’ll be sharing file systems and UID’s 
# across containers we need to match the operating systems.

FROM debian:jessie
MAINTAINER artosa

# NOTE: we set the UID here to the same one the Cloudbees Jenkins image uses 
# so we can match UIDs across containers, 
# which is essential if you want to preserve file permissions between the containers. 
# We also use the same home directory and bash settings.

RUN useradd -d "/var/jenkins_home" -u 1000 -m -s /bin/bash jenkins

RUN mkdir -p /var/log/jenkins
RUN chown -R jenkins:jenkins /var/log/jenkins

VOLUME ["/var/log/jenkins", "/var/jenkins_home"]

USER jenkins

CMD ["echo", "Data container for Jenkins"]




#Part 3: Add NGINX front end server, create new folder, and Dockerfile (jenkins-nginx):


FROM centos:centos7
MAINTAINER artosa

#Install NGINX with yum:
RUN yum -y update; yum clean all
RUN yum -y install http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm; yum -y makecache
RUN yum -y install nginx-1.8.0

#Cleanup some nginx default configs we don't need
RUN rm /etc/nginx/conf.d/default.conf
RUN rm /etc/nginx/conf.d/example_ssl.conf

#Copy some custom configs
COPY conf/jenkins.conf /etc/nginx/conf.d/jenkins.conf
COPY conf/nginx.conf /etc/nginx/nginx.conf

#Expose port 80
EXPOSE 80

#Make sure nginx is started
CMD ["nginx"]





#So we need some config files next, here's nginx.conf:

daemon off;
user  nginx;
worker_processes  2;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
    use epoll;
    accept_mutex off;
}

http {
    include       /etc/nginx/mime.types;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    client_max_body_size 300m;
    client_body_buffer_size 128k;

    gzip  on;
    gzip_http_version 1.0;
    gzip_comp_level 6;
    gzip_min_length 0;
    gzip_buffers 16 8k;
    gzip_proxied any;
    gzip_types text/plain text/css text/xml text/javascript application/xml application/xml+rss application/javascript application/json;
    gzip_disable "MSIE [1-6]\.";
    gzip_vary on;

    include /etc/nginx/conf.d/*.conf;
}


#Some docker notes:
daemon off;
We do this because by default calling “nginx” at the command line has NGINX run as a background daemon. That returns “exit 0” which causes Docker to think the process has stopped, and it shuts down the container. You’ll find this happens a lot with applications not designed to run in containers. Thankfully for NGINX this simple change solves the problem without a complex workaround.

worker_processes 2;
This is something I do with every NGINX I set up. You can leave this at 1 if you want. It’s really a “tune as you see fit” option. NGINX tuning is a topic for a post in its own right. I can’t tell you what’s right for you. Very roughly speaking, this is how many individual NGINX processes you have. The number of CPU’s you’ll allocate is a good guide. Hordes of NGINX specialists will say its more complicated than that. Certainly inside a Docker container you could debate what to do here.

use epoll;
accept_mutex off;
Turning epolling on is a handy tuning mechanism to use more efficient connection handling models. We turn off accept_mutex for speed, because we don’t mind the wasted resources at low connection request counts.

proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
So this is the second setting (after turning daemon off) that’s a must-have for Jenkins proxying. This sets the headers so that Jenkins can interpret the requests properly, which helps eliminate some warnings about improperly set headers.

client_max_body_size 300m;
client_body_buffer_size 128k;
You may or may not need this. Admittedly, 300 MBs is a large body size. However, we have users that upload files to our Jenkins server—some of which are just HPI plugins, while others are actual files. We set this to help them out.

GZIP on:
gzip on;
gzip_http_version 1.0;
gzip_comp_level 6;
gzip_min_length 0;
gzip_buffers 16 8k;
gzip_proxied any;
gzip_types text/plain text/css text/xml text/javascript application/xml application/xml+rss application/javascript application/json;
gzip_disable "MSIE [1-6]\.";
gzip_vary on;



#And here's jenkins.conf file:

server {
    listen       80;
    server_name  "";

    access_log off;

    location / {
        proxy_pass         http://jenkins-master:8080;

        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto http;
        proxy_max_temp_file_size 0;

        proxy_connect_timeout      150;
        proxy_send_timeout         100;
        proxy_read_timeout         100;

        proxy_buffer_size          8k;
        proxy_buffers              4 32k;
        proxy_busy_buffers_size    64k;
        proxy_temp_file_write_size 64k;	

    }

}

#Some notes on Ubuntu, it seems to be different
#To install nginx you can do sudo apt-get install nginx
#Instead of copying both files to under conf.d, you probably want to overwrite nginx.conf at upper level, and remove default file from sites in use folder
#Might be that user should be different for nginx in Ubuntu


#Time to build the image:

docker build -t myjenkinsnginx jenkins-nginx/.

#Make sure other two containers are running, then start nginx container with --link option
docker run --name=jenkins-data myjenkinsdata
docker stop jenkins-master
docker rm jenkins-master
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d myjenkins
docker run -p 80:80 --name=jenkins-nginx --link jenkins-master:jenkins-master -d myjenkinsnginx

#So what does the --link do? Makes sure domain name jenkins-master exists in nginx container and links to jenkins-master container

So it should work like this:
http://192.168.99.100/

#Now we can remove port 80 from Jenkins machine, now we need to shutdown nginx container too, since it's linked:

docker stop jenkins-nginx
docker stop jenkins-master
docker rm jenkins-nginx
docker rm jenkins-master
docker run -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d myjenkins
docker run -p 80:80 --name=jenkins-nginx --link jenkins-master:jenkins-master -d myjenkinsnginx




#Using Compose (Fig) to keep track of multi-container setup
- Start by creating docker-compose.yml file to project root
	- In it, you can define rules for nested docker builds, remember it's yaml!

#For example, docker-compose.yml could contain this:

jenkinsdata:
  build: jenkins-data

jenkinsmaster:
  build: jenkins-master
  volumes_from:
    - jenkinsdata
  ports:
    - "50000:50000"

jenkinsnginx:
  build: jenkins-nginx
  ports:
     - "80:80"
  links:
     - jenkinsmaster:jenkins-master    

#Remove old traces
docker stop jenkins-nginx
docker rm jenkins-nginx
docker stop jenkins-master
docker rm jenkins-master
docker rm jenkins-data


#To build and run, -d for daemon:

docker-compose build	
docker-compose up -d

#Here's hot to see what's running:
docker-compose ps

#To stop
docker-compose stop

#To remove everything:
docker-compose rm

#To remove all but data:
docker-compose rm jenkinsmaster jenkinsgninx

