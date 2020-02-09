##  Docker-machine setup  ##

### Docker-machine setup AWS : ###

#### Prerequisites : ####
    
	1. Linux machine installed awscli,docker-machine command line utilities and docker client.
	2. Aws - IAM user with programatic access and  ec2 fullaccess.
	3. Configure IAM credentials with aws-cli.
			  
			  
#### Install aws-cli : ####


	sudo apt-get update

	sudo apt install python-pip -y

	pip install awscli

	export PATH=$PATH:~/.local/bin

#### Install docker-machine : ####

	curl -L https://github.com/docker/machine/releases/download/v0.13.0/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine && \
	chmod +x /tmp/docker-machine && \
	sudo cp /tmp/docker-machine /usr/local/bin/docker-machine

#### Install docker : ####


	apt-get update
	sudo apt-get install \
	apt-transport-https \
	ca-certificates \
	curl \
	gnupg-agent \
	software-properties-common -y
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
	apt-get update
	sudo apt-key fingerprint 0EBFCD88
	sudo add-apt-repository \
	"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
	$(lsb_release -cs) \
	stable"
	sudo apt-get update
	sudo apt-get install docker-ce docker-ce-cli containerd.io -y
	docker -v

#### Configure the aws credentials : ####

	aws configure

Give aws accessID, secretkey, region, output type


#### Create docker machine using aws ec2 driver : ####

NOTE :  Base-machine is where awcli, docker-machine and docker-client has been installed.

Intially we need to run the  ' consul ' image to discovery process :

Run the below command in Base-machine

	docker run -d -p 8500:8500 progrium/consul -server -bootstrap -advertise=34.205.255.243 -ui-dir /ui

NOTE :  advertise ip should be consul IP (where cosul is installed machine)

Create docker-machine swarm manager :

NOTE : For swarm manager we need token, to generate certificates and this token will get by running the following command.
       Command : 
       
       docker run -d swarm create.
       
#### create the swarm-manager machine using token ###

	docker-machine create -d amazonec2 \
	--amazonec2-region us-east-1 \
	--amazonec2-instance-type "t2.large" \
	--swarm \
	--swarm-master \
	--swarm-discovery token://7aad6b0526507d4c0f5349ac2a5af73ac3f0d5ee597924f17d9f8f5caa82c5fe \
	--swarm-strategy spread \
	--swarm-opt heartbeat=5s \
	swarm-manager


       
#### Create docker-machine nodes : ####

	docker-machine create -d amazonec2 \
	--amazonec2-region us-east-1 \
	--amazonec2-instance-type "t2.large" \
	node-01

	docker-machine create -d amazonec2 \
	--amazonec2-region us-east-1 \
	--amazonec2-instance-type "t2.large" \
	node-02

 	
#### After create the docker-machine swarm manager and nodes : ####

1. we need to login to swarm manager and stop the running manager docker container and join docker container .

		docker-machine ssh swarm-manager
		 sudo docker stop $(docker ps -q) 
	  
2. Create a new docker container with consul discovery method 
  
NOTE : Change the advertise IP address with swarm-manager IP.
       Change the consul IP address with consul server IP.  

	sudo docker run -d -p 3376:3376 -v /etc/docker/:/certs:ro swarm manage --tlsverify --tlscacert=/certs/ca.pem --tlscert=/certs/server.pem --tlskey=/certs/server-key.pem -H tcp://0.0.0.0:3376 --strategy spread --advertise 34.200.230.45:3376 consul://34.205.255.243:8500
	
	sudo docker ps 

docker swarm container should be up and running with 3376 port


	
3. Exit from the swarm-manager and Login to nodes 
 
	   docker-machine ssh node01
   
   stop the running join docker container
   
	   sudo docker stop $(docker ps -q)
   
Note : Change below command advertise IP address with node01 IP 
       Change consul IP address with consul server IP

	sudo docker run -d swarm join --advertise 34.205.50.43:2376 consul://34.205.255.243:8500
	sudo docker ps
docker swarm container should be up and running with 2376 port

4. If you want to add more nodes repeat the steps from 3. 

## The cluster creation has been done ##


#### Now test the cluster: ####

Login to base-machine and switch to root user 

	mkdir -p ~/.docker
	cd ~/.docker

copy the swarm-manager certificates to .docker folder.

	cp -r /home/ubuntu/.docker/machine/machines/swarm-manager/*.pem* ~/.docker/

export the env for connecting the swarm-manager 

NOTE : change the tcp IP address with swarm-manager IP and this command will work for current terminal only

	export DOCKER_HOST=tcp://34.200.230.45:3376 DOCKER_TLS_VERIFY=1

Check the swarm API version with below command 

	docker version --format '{{.Server.Version}}'

Now run the a test conatiner in cluster :

	docker run -d ngnix 
	docker ps

Finaly login to docker nodes and check the nginx conatiner should be up and running in any one of the node and should not be run in swarm-manager.
