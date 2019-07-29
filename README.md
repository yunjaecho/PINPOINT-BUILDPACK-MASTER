# Pinpoint Buildpack 

## Purpose of this buildpack.
[Pinpoint](https://naver.github.io/pinpoint/index.html) is an APM (Application Performance Management) tool for large-scale distributed systems written in Java / PHP. Inspired by Dapper, Pinpoint provides a solution to help analyze the overall structure of the system and how components within them are interconnected by tracing transactions across distributed applications.
This buildpack installs Pinpoint agent and configuration to your app automatically. and intended to seperate pinpoint buildpack from java_buildpack_offline using [muilti-buildpack framework](https://docs.cloudfoundry.org/buildpacks/understand-buildpacks.html), so that you can keep updating java_buildpack_offline regardless of pinpoint buildpack version.

## How it works
This is a non-final buildpack for Cloud Foundry that provides integration with Pinpoint agent(https://naver.github.io/pinpoint)
This buildpack works with final buildpack that supports multi buildpack such as [java-buildpack](https://github.com/cloudfoundry/java-buildpack/blob/master/docs/framework-multi_buildpack.md#multiple-buildpack-integration-api)
1. you need to specify this buildpack for non-final buildpack when you push your app to cloud foundry as following:
```
cf push -f manifest.yml -b https://github.com/myminseok/pinpoint-buildpack.git -b java_buildpack_offline -p build/libs/spring-music.jar
```
2. cloud foundry stages a container image by putting  downloading pinpoint-buildpack from the location you specified.
3. pinpoint-buildpack will download two files to staging container 1) pinpoint-agent-1.8.2.tar.gz 2) pinpoint.config from the location you specified during the step1
4. untar pinpoint-agent-1.8.2.tar.gz under /home/vcap/deps/0/pinpoint
5. place pinpoint.config  /home/vcap/deps/0/pinpoint
6. set JAVA_OPTS Envirionment variables for pinpoint agent in /home/vcap/deps/0/config.yml. this file will be used by java_buildpack_offline buildpack to run app. config.yml has pinpoint metadata such as AgentId, Application Name, AGENT_HOME, agent jar path.
7. java_buildpack_offline start to staging runtime env for the apps.
8. prepare runtime conmmand using /home/vcap/deps/0/config.yml.

9. start  app containers using the staged container. when app runs, it will emits metrics to pinpoint server.
10. in pinpoint server, each container will have Id with <APPLICATION_NAME>-<INSTANCE_INDEX> under <APPLICATION_NAME> view.


## deploy sample app

1. prepare application jar
```sh
git clone  https://github.com/myminseok/spring-music
cd spring-music
./gradlew clean assemble
```
2. prepare manifest.yml

```
---
applications:
- name: spring-music
  memory: 1G
  buildpacks:
  - https://github.com/myminseok/pinpoint-buildpack
  - java_buildpack_offline
  random-route: true
  #path: build/libs/spring-music.jar
  path: ./spring-music.jar
  env:
    PINPOINT_AGENT_PACKAGE_DOWNLOAD_URL: https://github.com/naver/pinpoint/releases/download/1.8.2/pinpoint-agent-1.8.2.tar.gz
    PINPOINT_CONFIG_URL: https://raw.githubusercontent.com/myminseok/pinpoint_agent_repo/master/pinpoint.config-1.8.2
```
- buildpacks: specify pinpoint-buildpack and java_buildpack_offline
- PINPOINT_AGENT_PACKAGE_DOWNLOAD_URL: pinpoint-agent url to download url(https://..../pinpoint-agent-1.7.4-SNAPSHOT.tar.gz). see https://github.com/naver/pinpoint/releases. (optional, default in buildpack : https://github.com/myminseok/pinpoint_agent_repo/blob/master/pinpoint-agent-1.8.2-SNAPSHOT.tar.gz?raw=true)
- PINPOINT_CONFIG_URL: configuraton for pinpoint. profiler.collector.ip should point the pinpoint server. (optional, default in buildpack: pinpoint.config packaged in PINPOINT_AGENT_PACKAGE_DOWNLOAD_URL)

3. push apps
```
cf push -f manifest.yml

cf logs spring-music
```



## Setup pinpoint server (docker on ubuntu)
0. [requirements](https://docs.docker.com/compose/compose-file/)
- Docker Engine release: 18.02.0+
- Docker compose: 3.6+

1. [install docker-compose](https://docs.docker.com/compose/install/)
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

docker-compose --version
=> docker-compose version 1.23.2, build 1110ad01

```

2. [run pinpoint server(docker)](https://github.com/naver/pinpoint-docker/releases)
```
wget https://github.com/naver/pinpoint-docker/archive/1.8.2.tar.gz

tar xf  1.8.2.tar.gz
cd pinpoint-1.8.2
sudo docker-compose up

```

3. open pinpoint web
```
open http://<pinpoint-server-ip>:8079
```  


## How to build buildpack(offline)
- prepare your pinpoint agent zip and pinpoint.config. refer to (sample repo](https://github.com/myminseok/pinpoint_agent_repo)
- edit buildpack configuration.
```
git clone https://github.com/myminseok/pinpoint-buildpack
cd pinpoint-buildpack

vi lib/config.yml . 
vi bin/supply

./upload

```
  
### Reference.
- https://naver.github.io/pinpoint/1.7.3/installation.html
- https://github.com/naver/pinpoint/blob/master/doc/installation.md
- See https://docs.cloudfoundry.org/buildpacks/understand-buildpacks.html for buildpack basics. This is an intermediate buildpack using only the bin/supply script.

This buildpack is inspired by https://github.com/cf-platform-eng/eureka-registrar-sidecar


