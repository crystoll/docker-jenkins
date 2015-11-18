
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