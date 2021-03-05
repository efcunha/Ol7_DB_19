Just some Dockerfiles I'm playing around with. Some will be simple and stupid. 
Others might be a bit more useful to look at if you are learning about Docker.

Feel free to copy or fork this stuff. It's unlikely I'll accept pull requests as it's just my playground.

# Prerequisites

Download Softwares Installation links:
```
https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html
https://www.oracle.com/database/technologies/oracle18c-linux-180000-downloads.html
https://www.oracle.com/tools/downloads/apex-downloads.html
https://www.oracle.com/database/technologies/appdev/rest-data-services-downloads.html
https://www.oracle.com/tools/downloads/sqlcl-downloads.html
https://adoptopenjdk.net/releases.html
http://mirror.nbtelecom.com.br/apache/tomcat/tomcat-9/v9.0.37/bin/apache-tomcat-9.0.37.tar.gz
```
# Oracle Database 19 on Docker SWARM

# Host Setup

From the user "root" create the directories where you want to store the persistent volumes used by the containers. I'm going to create them under the "docker_user" user's home, but they could be anywhere.
```
mkdir -p /u01/volumes/ol7_19_ords_tomcat
mkdir -p /u01/volumes/ol7_19_ords_db

```
As the "root" user create a new group with a specific group ID, which will be used for group ownership inside the container and on the host file. Below you can see we've altered the group ownership and permissions on the directories, including the sticky bit for the group permissions. We've the added the "docker_user" user to the "docker_fg" group.
```
groupadd -g 1042 docker_fg
chown -R :docker_fg /u01/volumes
chmod -R 775 /u01/volumes
chmod -R g+s /u01/volumes
usermod -aG docker_fg docker_user
```
# Image/Container Setup

For this to work we have to make sure the volume defined in the container has the same group permissions. For the ORDS container image build we create the same group we did on the host, and make the "tomcat" user part of that group.
```
groupadd -g 1042 docker_fg
useradd tomcat -G docker_fg
```
Later in the image build we create the CATALINA_BASE location and set the permissions as we did on the host.
```
mkdir -p ${CATALINA_BASE}
chown :docker_fg ${CATALINA_BASE}
chmod 775 ${CATALINA_BASE}
chmod g+s ${CATALINA_BASE}
```
When we run the container, both the host file and container have a group (docker_fg) with the same ID (1042), and the group ownership of the directory is configured to use it.

We are now able to access the directories from within the container, and from the "docker_user" user on the hosts file system, rather than being forced to use the "root" user to access it.

Directory contents when software is included 
```
$ tree
.
├── Dockerfile
├── README.md
├── compose
├── ords
├── database/ol7_19/scripts
│   ├── healthcheck.sh
│   └── start.sh
├── database/ol7_19/software
│   ├── apex_20.1_en.zip
│   ├── LINUX.X64_193000_db_home.zip or LINUX.X64_190000_db_home.zip
│   └── put_software_here.txt
└── swarm
$
```
# Oracle REST Data Services (ORDS) on Docker

Directory contents when software is included.
```
$ tree
.
├── Dockerfile
├── README.md
├── compose
├── database
├── ords/ol7_ords/scripts
│   ├── healthcheck.sh
│   ├── install_os_packages.sh
│   ├── ords_software_installation.sh
│   ├── server.xml
│   └── start.sh
├── ords/ol7_ords/software
│   ├── apache-tomcat-9.0.37.tar.gz
│   ├── apex_20.1_en.zip
│   ├── OpenJDK11U-jdk_x64_linux_hotspot_11.0.8_10.tar.gz
│   ├── ords-20.2.0.178.1804.zip
│   ├── put_software_here.txt
│   └── sqlcl-20.2.0.174.1557.zip
└── swarm
$
```

In order for this to work, without adjustment, you will need to perform the following prerequisites.

You will need to place the relevant installation media under the software directories of 
the "ol7_183" or "ol7_19" database build and the "ol7_ords" build.

# Build the images.
```
cd ~/Ol7_DB_19/database/ol7_19
docker build -t ol7_19:latest .

# Build ORDS image
cd ~/Ol7_DB_19/ords/ol7_ords
docker build -t ol7_ords:latest .
```
The example also uses a Portainer image, but that is downloaded automatically from Docker Hub when required.

Once you've completed these prerequisites without errors, we can move on.

# Stack Definition

If you've used Docker Compose, the definition of a stack will look very familar, as it uses a compose file. 
Some examples still use a file name called "docker-compose.yml", while others use the name "docker-stack.yml" for the file name. 
I prefer the latter, as there are some options which are valid for Docker Compose, which are not valid for a stack definition.

For this article I'm using the "docker-stack.yml" file found here. This file defines a three services.

db : A service to run an Oracle database container.
ords : A service to run two Oracle REST Data Services (ORDS) containers.
portainer : A service to run a Portainer container.
The "docker-stack.yml" contains the following entry for the ORDS service, which defines how is should be deployed. 
In this case we are expecting 2 containers, which the swarm will restart on failure, with each being limited to 1 CPU.

    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "1"

The rest of the definition matches what you would see in a regular Docker Compose file, minus the build instructions. 
If you reference any compose-specific parameters, they will be ignored and a warming displayed on screen.

# Deploy Stack

We deploy a stack to the swarm using the docker stack deploy command, referencing the "docker-stack.yml" file using the "--compose-file" or "-c" flag.
```
$ cd ~//Ol7_DB_19/swarm/ol7_19_ords

$ docker stack deploy --compose-file ./docker-stack.yml ords-stack
Creating network ords-stack_ordsnet
Creating service ords-stack_ords
Creating service ords-stack_db
$
The following commands give us some information about the stack. We can check what stacks are in the swarm using the docker stack ls command.

$ docker stack ls
NAME                SERVICES
ords-stack          3
$
We can see the services that are running, including the number of replicates of each, using the docker service ls command.

$ docker service ls
ID                  NAME                   MODE                REPLICAS            IMAGE                        PORTS
3yy4pncrars5        ords-stack_db          replicated          1/1                 ol7_183:latest               *:1521->1521/tcp
0eo2a8pi91tg        ords-stack_ords        replicated          2/2                 ol7_ords:latest              *:8080->8080/tcp, *:8443->8443/tcp
u7qnhf4l2tyd        ords-stack_portainer   replicated          1/1                 portainer/portainer:latest   *:9000->9000/tcp
$
We can display information about the processes running in a stack using the docker stack ps {stack name} command.

$ docker stack ps ords-stack
ID                  NAME                     IMAGE                        NODE                    DESIRED STATE       CURRENT STATE            ERROR               PORTS
j4rpczp0k6ch        ords-stack_portainer.1   portainer/portainer:latest   localhost.localdomain   Running             Running 5 minutes ago
jlrmo1g0yqta        ords-stack_db.1          ol7_183:latest               localhost.localdomain   Running             Running 17 seconds ago
rkvub2qez8ub        ords-stack_ords.1        ol7_ords:latest              localhost.localdomain   Running             Running 4 minutes ago
43usasqwjn6s        ords-stack_ords.2        ol7_ords:latest              localhost.localdomain   Running             Running 4 minutes ago
$
The docker ps command can still be used in the normal way of course.

$ docker ps -a
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                   PORTS                NAMES
fe92be4d84f8        ol7_183:latest               "/bin/sh -c 'exec ${…"   7 minutes ago       Up 6 minutes (healthy)   1521/tcp, 5500/tcp   ords-stack_db.1.jlrmo1g0yqtagbjbz3rn7gyff
6991979026f1        ol7_ords:latest              "/bin/sh -c 'exec ${…"   7 minutes ago       Up 7 minutes (healthy)   8080/tcp, 8443/tcp   ords-stack_ords.2.43usasqwjn6s9wl03y3uyzdvx
ca3b6a58ebe2        ol7_ords:latest              "/bin/sh -c 'exec ${…"   7 minutes ago       Up 7 minutes (healthy)   8080/tcp, 8443/tcp   ords-stack_ords.1.rkvub2qez8ubtxewibuiab0ep
$
```
# Scale a Service

We can scale up or down the replicas for a specific service using the docker service scale command. 
The following command increases the number of replicates from 2 to 5 for the ORDS service, 
using the service name listed with the docker service ls command. 
The output from the command shows the additional containers starting.
```
$ docker service scale ords-stack_ords=5
ords-stack_ords scaled to 5
overall progress: 5 out of 5 tasks
1/5: running   [==================================================>]
2/5: running   [==================================================>]
3/5: running   [==================================================>]
4/5: running   [==================================================>]
5/5: running   [==================================================>]
verify: Service converged
$

$ docker stack ps ords-stack
ID                  NAME                     IMAGE                        NODE                    DESIRED STATE       CURRENT STATE                ERROR               PORTS
jlrmo1g0yqta        ords-stack_db.1          ol7_183:latest               localhost.localdomain   Running             Running 9 minutes ago
rkvub2qez8ub        ords-stack_ords.1        ol7_ords:latest              localhost.localdomain   Running             Running 13 minutes ago
43usasqwjn6s        ords-stack_ords.2        ol7_ords:latest              localhost.localdomain   Running             Running 13 minutes ago
rnw0i26gudqn        ords-stack_ords.3        ol7_ords:latest              localhost.localdomain   Running             Running about a minute ago
a64bj66nv9rh        ords-stack_ords.4        ol7_ords:latest              localhost.localdomain   Running             Running about a minute ago
kdh39395zdc8        ords-stack_ords.5        ol7_ords:latest              localhost.localdomain   Running             Running about a minute ago
$

We can scale down the service also.

$ docker service scale ords-stack_ords=2
ords-stack_ords scaled to 2
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged
$

$ docker stack ps ords-stack
ID                  NAME                     IMAGE                        NODE                    DESIRED STATE       CURRENT STATE            ERROR               PORTS
jlrmo1g0yqta        ords-stack_db.1          ol7_183:latest               localhost.localdomain   Running             Running 11 minutes ago
rkvub2qez8ub        ords-stack_ords.1        ol7_ords:latest              localhost.localdomain   Running             Running 15 minutes ago
43usasqwjn6s        ords-stack_ords.2        ol7_ords:latest              localhost.localdomain   Running             Running 15 minutes ago
$
```
# Remove Stack

The docker stack rm command will remove the specified stack.
```
$ docker stack rm ords-stack
Removing service ords-stack_db
Removing service ords-stack_ords
Removing network ords-stack_ordsnet
$

Depending on the container shutdowns, it can take some time to complete, 
so you might want to check the status using the docker stack ps command, 
until all the processes are gone.

$ docker stack rm ords-stack
Removing service ords-stack_db
Removing service ords-stack_ords
Removing network ords-stack_ordsnet
$

$ docker stack ps ords-stack
ID                  NAME                          IMAGE               NODE                    DESIRED STATE       CURRENT STATE           ERROR               PORTS
jlrmo1g0yqta        3yy4pncrars5wk9bvucs1nf2l.1   ol7_183:latest      localhost.localdomain   Remove              Running 5 seconds ago
$

$ docker stack ps ords-stack
nothing found in stack: ords-stack
$
```
# Leave Swarm

A machine can leave a swarm using the docker swarm leave command. 
The "-f" flag allows the swarm manager to leave the swarm.
```
$ docker swarm leave -f
Node left the swarm.
$
```
# Considerations

Just some things to consider when using Docker Swarm.

- Swarm assumes all containers can be run on any node in the cluster, unless expressly restricted using a placement constraint. 
In the example used here, the DB and Portainer services are restricted to the Swarm manager node using a constraint. 
If the services are accessing any external resources, like file systems, they need to be present on all nodes. 
It's common to use NFS to allow file systems to we shared between nodes in the cluster.

- Most of what we did with the stack definition can be performed manually from the command line, 
but it makes sense to define applications in a "docker-stack.yml" file, so it can be checked into version control.
