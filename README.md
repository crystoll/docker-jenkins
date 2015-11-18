This is a simple sample project to get started with Dockerifying Jenkins

This is based on article series by RIOT games at http://engineering.riotgames.com/news/thinking-inside-container - with some modifications for personal use, and more notes.

To build and run all, you need to have docker-tools properly installed, and docker-machine running.

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


Jenkins is running at http://[dockermachineipaddress]/jenkins

Its jobs and settings are persisted under jenkins-data, and it has got nginx server in front to redirect port 80 to 8080 (and handle any other stuff that's convenient)

It could be easily supplemented by other tools as separate containers, here's suggestions:

- Sonar quality analysis
- Selenium grid for running e2e tests in other machine than Jenkins
- Separate database instance for Sonar

