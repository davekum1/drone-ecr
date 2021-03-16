# Automating CI (Build) with Drone
Drone is open source CI tools which can be used for self-service continuous integration for development team. Compare to Jenkins that requires Jenkins pipeline need to be build, Drone uses drone.yml as template (IaaS concept).

## Requirement
In order to run Drone, you will need some environment to deploy Drone server and runner. Example below are build within Drone server/runner running in AWS EC2.

## Environment preparation
- Provision EC2 with public IP enabled 
- Make sure you set security group to allow port 80 (http) and 22 (SSH)
- Once your VM up and running, make sure the IP is reachable from internet
- Make sure Docker is installedd in EC2

## Installing Drone
You will need to install both Drone server and runner in the EC2 above via docker container. We could use Kubernetes as container orchestration, but this example will install drone server and drone runner as container in the EC2 (without orchestration framework).

First, you need to pull drone server from Docker public image
```
docker pull drone/drone:1
```
You will need to pass in some parameters when running dockers. Please refer to table below for variables name and how to obtain them.

| Variable | Value |
| ------ | ------ |
| DRONE_GITHUB_CLIENT_ID | You need to create OAuth app in Github. Should be under Settings-Developer Settings-OAuth Apps in Github. For how to do it, please see from [this drone tutorial](https://www.youtube.com/watch?v=kZmOCLCpvmk&t=106s) |
| DRONE_GITHUB_CLIENT_SECRET | You need to create OAuth app in Github. Should be under Settings-Developer Settings-OAuth Apps in Github. For how to do it, please see from [this drone tutorial](https://www.youtube.com/watch?v=kZmOCLCpvmk&t=106s) |
| DRONE_RPC_SECRET | This can be generated from VM using command "openssl rand -hex 16" |
| DRONE_SERVER_HOST | In this case, it is EC2 public IP |
| DRONE_SERVER_PROTO | http or https |


Run drone server as below
```
 docker run \
  --volume=/var/lib/drone:/data \
  --env=DRONE_GITHUB_CLIENT_ID=<GITHUB Client Id from table above> \
  --env=DRONE_GITHUB_CLIENT_SECRET=<GITHUB Client Id from table above> \
  --env=DRONE_RPC_SECRET=<RPC_Secret from table above> \
  --env=DRONE_SERVER_HOST=<EC2 public IP address> \
  --env=DRONE_SERVER_PROTO=http \
  --env=DRONE_USER_CREATE=username:<github userId>,admin:true \
  --publish=80:80 \
  --publish=443:443 \
  --restart=always \
  --detach=true \
  --name=drone \
  drone/drone:1
```
Pull drone runner
```
docker pull drone/drone-runner-docker:1
```
Run drone runner as below
```
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e DRONE_RPC_PROTO=http \
  -e DRONE_RPC_HOST=<EC2 public IP Address> \
  -e DRONE_RPC_SECRET=<RPC Secret from table above> \
  -e DRONE_RUNNER_CAPACITY=2 \
  -p 3000:3000 \
  --restart always \
  --name runner \
  drone/drone-runner-docker:1
```

More detail about Drone server and runner installation, please refer to [Drone documentation](https://docs.drone.io/server/provider/github/)

## Understanding Drone yaml
The power of Drone is everything can be configured through codes (IaaS), in this case, it refers to drone.yml file. Example provided in this repo is to build docker image from simple Go application and then push to AWS ECR.

Some tips while configuring ECR for Drone
- EC2 where drone runs MUST HAVE IAM access to AWS ECR. You can create user with permission policies to AmazonEC2ContainerRegistry. Set this access key/secretkey to EC2 using aws configure command
- Use Drone secret while configuring aws accesskey and secretkey. Do not check in both of the value to public git repository

For more detail information about setting up ECR for Drone, please check [at drone documentaton](http://plugins.drone.io/drone-plugins/drone-ecr/)

## Running Drone
Once everything setup, all you need to do is to commit and push your code to Git. Drone will automatically activity feeds, build the code, create docker container, and push to AWS ECR