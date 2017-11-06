# deors.demos.microservices.docs

## microservices with Spring Boot and Spring Cloud

this repository contains the step-by-step instructions to recreate the microservices with Spring Boot and Spring Cloud demonstration

this demo is organised in iterations, starting from the basics and building up in complexity and features along the way

NOTE: the following instructions are created on a Windows machine, hence some commands may need slight adjustments when working on Linux/OSX, e.g. replace %ENV\_VAR% by ${ENV\_VAR}

## iteration 1) the basics

### 1.1) set up the configuration store

the configuration store is a repository where microservice settings are stored, and accessible for microservice initialisation at boot time

create and change to a directory for the project

    mkdir %HOME%\microservices\deors.demos.microservices.configstore
    cd %HOME%\microservices\deors.demos.microservices.configstore

create file application.properties; these settings are common to all microservices

    debug = true
    spring.jpa.generate-ddl = true
    management.security.enabled = false

the third configuration setting will disable security for actuator endpoints, which allows for remote operations of running applications; disabling security in this manner should be done only when public access to those endpoints is restricted externally (for example through the web server or reverse proxy); never expose actuator endpoints publicly and unsecurely!

create file eurekaservice.properties

    server.port = ${PORT:7878}
    eureka.client.register-with-eureka = false
    eureka.client.fetch-registry = false
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/

create file hystrixservice.properties

    server.port = ${PORT:7979}
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/

create file turbineservice.properties

    server.port = ${PORT:8989}
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/

create file bookrecservice.properties

    server.port = ${PORT:8080}
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/

create file bookrecedgeservice.properties

    server.port = ${PORT:8181}
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/
    ribbon.eureka.enabled = true
    defaultBook = this is the default recommendation: Book {id=-1, title='robots of dawn', author='isaac asimov'}

initialise the Git repository

    git init
    git add .
    git commit -m "initial configuration"

publish it online (i.e. GitHub, replace with your own repository)

    git remote add origin https://github.com/deors/deors.demos.microservices.configstore.git
    git push origin master

### 1.2) set up the configuration service

the configuration service, powered by Spring Cloud Config Server, is the microservice that will provide every other microservice in the system with the configuration settings they need at boot time

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: configservice
    depedencies:
        actuator
        config server

the actuator depedency, when added, enables useful endpoints to facilitate application operations

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\configservice

add the application name and ensure config store location is properly set;
edit src\main\resources\application.properties

    server.port = ${PORT:8888}
    spring.application.name = configservice
    spring.cloud.config.server.git.uri = ${CONFIG_REPO_URL:https://github.com/deors/deors.demos.microservices.configstore.git}

configure config server to start automatically;
edit src\main\java\deors\demos\microservices\configservice\ConfigserviceApplication.java

add class annotation

    @org.springframework.cloud.config.server.EnableConfigServer

### 1.3) set up the service registry and discovery service (Eureka)

the service registry and discovery service, powered by Spring Cloud and Netflix Eureka, is the microservice that will enable every other microservice in the system to register 'where' they are physically located, so others can discover them and interact with them

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: eurekaservice
    depedencies:
        actuator
        config client
        eureka server

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\eurekaservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = eurekaservice
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure eureka server to start automatically;
edit src\main\java\deors\demos\microservices\eurekaservice\Application.java

add class annotation

    @org.springframework.cloud.netflix.eureka.server.EnableEurekaServer

### 1.4) set up the circuit breaker dashboard service (Hystrix)

the circuit breaker dashboard, powered by Spring Cloud and Netflix Hystrix, will provide devs and ops teams with real-time views about service calls performance and failures including for example which of them are experimenting repeated failures causing circuits to 'open' and calls to default to a safe method to keep them up and running smoothly, specially without cascading failures across services and causing major downtimes

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: hystrixservice
    depedencies:
        actuator
        config client
        eureka discovery
        hystrix dashboard

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\hystrixservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = hystrixservice
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure hystrix dashboard to start automatically;
edit src\main\java\deors\demos\microservices\hystrixservice\HystrixserviceApplication.java

add class annotations

    @org.springframework.cloud.client.discovery.EnableDiscoveryClient
    @org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard

### 1.5) set up the circuit breaker metric aggregation service (Turbine Stream)

while Hystrix does a very good job to route service calls only to healthy service instances, the reporting in Hystrix dashboard is not aggregating observations from all instances in a service

within Swarm the result is even less satisfactory, as the dashboard is showing metrics from only one instance at a time, which is selected by Swarm load balancer in a way that is not predictable for the user

to change that behaviour, a circuit breaker metric aggregation service, powered by Netflix Turbine with Spring Cloud Stream service will be added to the stack; each Hystrix client will send metrics to Turbine, and the Hystrix dashboard will connect with Turbine service to obtain the circuit breaker aggregated metrics

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: turbineservice
    depedencies:
        actuator
        config client
        eureka discovery
        turbine stream
        stream rabbit

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\turbineservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = turbineservice
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure Turbine and Stream to start automatically;
edit src\main\java\deors\demos\microservices\turbineservice\TurbineserviceApplication.java

add class annotation

    @org.springframework.cloud.client.discovery.EnableDiscoveryClient
    @org.springframework.cloud.netflix.turbine.stream.EnableTurbineStream

### 1.6) set up the book recommendation service

this is the first microservice with actual functionality on our problem domain; bookrec is the service which provides methods to query, create, update and remove Book entities from the data store

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: bookrecservice
    depedencies:
        actuator
        config client
        eureka discovery
        web
        rest repositories
        rest repositories hal browser
        hateoas
        jpa
        h2

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\bookrecservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = bookrecservice
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure service to be discoverable;
edit src\main\java\deors\demos\microservices\bookrecservice\BookrecserviceApplication.java

add class annotation

    @org.springframework.cloud.client.discovery.EnableDiscoveryClient

create Book entity (in IDE)

    @Entity
    public class Book {
        @Id
        @GeneratedValue
        private Long id;
        private String title;
        private String author;
    }

generate bean constructors, getters, setters and toString method

create BookRepository

    @RepositoryRestResource
    public interface BookRepository extends CrudRepository<Book, Long> {
        @Query("select b from Book b order by RAND()")
        List<Book> getBooksRandomOrder();
    }

create BookController

    @RestController
    public class BookController {
        @Autowired
        private BookRepository bookRepository;

        @RequestMapping("/bookrec")
        public String getBookRecommendation() throws UnknownHostException {
            return "the host in IP: "
                    + InetAddress.getLocalHost().getHostAddress()
                    + " recommends this book: "
                    + bookRepository.getBooksRandomOrder().get(0);
        }
    }

add some test data (in IDE, create src/main/resources/import.sql)

    insert into book(id, title, author) values (1, 'second foundation', 'isaac asimov')
    insert into book(id, title, author) values (2, 'speaker for the dead', 'orson scott card')
    insert into book(id, title, author) values (3, 'the player of games', 'iain m. banks')
    insert into book(id, title, author) values (4, 'the lord of the rings', 'j.r.r. tolkien')
    insert into book(id, title, author) values (5, 'the warrior apprentice', 'lois mcmaster bujold')
    insert into book(id, title, author) values (6, 'blood of elves', 'andrzej sapkowski')
    insert into book(id, title, author) values (7, 'harry potter and the prisoner of azkaban', 'j.k. rowling')
    insert into book(id, title, author) values (8, '2010: odyssey two', 'arthur c. clarke')
    insert into book(id, title, author) values (9, 'starship troopers', 'robert a. heinlein')

### 1.7) set up the book recommendation edge service

the bookrec edge service is used by clients to interact with bookrec service, which should be not exposed directly to clients

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: bookrecedgeservice
    depedencies:
        actuator
        config client
        eureka discovery
        hystrix
        ribbon
        web
        rest repositories
        rest repositories hal browser
        hateoas

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\bookrecedgeservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = bookrecedgeservice
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure service to be discoverable (Eureka) and to use circuit breaker (Hystrix);
edit src\main\java\deors\demos\microservices\bookrecedgeservice\BookrecedgeserviceApplication.java

add class annotations

    @org.springframework.cloud.client.discovery.EnableDiscoveryClient
    @org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker

add restTemplate() method to enable load balancing (Ribbon) when calling bookrecservice

    @Bean
    @org.springframework.cloud.client.loadbalancer.LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

create Book bean (in IDE)

    public class Book {
        private Long id;
        private String title;
        private String author;
    }

generate bean constructors (one with three properties), getters, setters and toString method

create BookController for edge service

    @RestController
    class BookController {
        @Autowired
        RestTemplate restTemplate;

        @Value("${defaultBook}")
        private String defaultBook;

        @RequestMapping("/bookrecedge")
        @com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand(fallbackMethod = "getDefaultBook")
        public String getBookRecommendation() {
            return restTemplate.getForObject("http://bookrecservice/bookrec", String.class);
        }

        public String getDefaultBook() {
            return defaultBook;
        }
    }

### 1.8) run services locally

services are now ready to be executed locally, using the sensible default configuration settings and the embedded runtimes provided by Spring Boot

each service will be run by executing this command in the project folder:

    mvn spring-boot:run

### 1.9) test services locally

the services should all be available at the defined ports in the local host

access the config service through one the actuator endpoints (it is currently unsecured)

    http://localhost:8888/env

check that config service is capable of returning the right configuration for some of the services

    http://localhost:8888/bookrecservice/default
    http://localhost:8888/eurekaservice/default

check that eureka service is up and the book recommendation service and edge service are registered

    http://localhost:7878/

check that hystrix service is up and running

    http://localhost:7979/hystrix

check that turbine service is up and running

    http://localhost:8989/

access the HAL browser on the book recommendation service

    http://localhost:8080/

access the book recommendation service

    http://localhost:8080/bookrec

access the HAL browser on the book recommendation edge service

    http://localhost:8181/

access the book recommendation edge service

    http://localhost:8181/bookrecedge

stop the book recommendation service, and access the book recommendation edge service again;
the default recommended book should be returned instead but the application keeps working

go back to hystrix dashboard and start monitoring the book recommendation edge service;
registering this URL in the dashboard (and optionally configuring the delay and page title)

     http://localhost:8181/hystrix.stream

once registered, try again to access the edge service, with and without the inner service up and running, and experiment how thresholds (number of errors in a short period of time) impact the opening and closing of the circuit between the inner and the edge service

## iteration 2) preparing for Docker and Swarm

### 2.1) set up Swarm

this demo assumes that an existing Docker Swarm is available; the following instructions will show how to create a simple one in VirtualBox, in the case that a swarm is not already available

the swarm will be formed by three manager nodes, and three worker nodes, named:

    docker-swarm-manager-1
    docker-swarm-manager-2
    docker-swarm-manager-3
    docker-swarm-worker-1
    docker-swarm-worker-2
    docker-swarm-worker-3

the machines will be deployed in its own network:

    192.168.66.1/24

being the first IP in DHCP pool:

    192.168.66.100

to create each machine in VirtualBox, this is the command that was used

    docker-machine create --driver virtualbox --virtualbox-cpu-count 1 --virtualbox-memory 1024 --virtualbox-hostonly-cidr "192.168.66.1/24" <docker-machine-name>

before beginning to configure the swarm, set the environment to point to the first machine

when in Windows

    @FOR /f "tokens=*" %i IN ('docker-machine env docker-swarm-manager-1') DO @%i

when in Linux

    eval $(docker-machine env docker-swarm-manager-1)

next, to initialise a swarm the following command is used

    docker swarm init --advertise-addr 192.168.66.100

upon initialisation, the swarm exposes two tokens, one to add new manager nodes, one to add new worker nodes;
the commands needed to get the tokens are these ones

    docker swarm join-token manager -q
    docker swarm join-token worker -q

with the tokens at hand, just change the environment to point to each machine, every manager and worker nodes

when in Windows

    @FOR /f "tokens=*" %i IN ('docker-machine env <docker-machine-name>') DO @%i

when in Linux

    eval $(docker-machine env <docker-machine-name>)

and use the swarm join command

    docker swarm join --token <manager-or-worker-token> 192.168.66.100:2377

once it is ready, the swarm can be stopped with the following command

    docker-machine stop docker-swarm-manager-1 docker-swarm-manager-2 docker-swarm-manager-3 docker-swarm-worker-1 docker-swarm-worker-2 docker-swarm-worker-3

and to start it again

    docker-machine start docker-swarm-manager-1 docker-swarm-manager-2 docker-swarm-manager-3 docker-swarm-worker-1 docker-swarm-worker-2 docker-swarm-worker-3

### 2.2) update Eureka configuration to leverage internal Swarm network

when services are deployed inside Docker Swarm, there are multiple networks active in the running container; to be able to use correctly the client-side load balancing, each running instance must register in Eureka with the IP address corresponding to the internal network (and not the ingress network)

move to bookrecservice folder, edit bootstrap.properties and add the following lines

    spring.cloud.inetutils.preferredNetworks[0] = 192.168
    spring.cloud.inetutils.preferredNetworks[1] = 10.0

also move to bookrecedgeservice folder, edit bootstrap.properties and add the same lines

    spring.cloud.inetutils.preferredNetworks[0] = 192.168
    spring.cloud.inetutils.preferredNetworks[1] = 10.0

finally, do the same for eurekaservice and hystrixservice; in general, this must be done in every service which is going to register in Eureka

with these changes, both services will register in Eureka with the right IP address, both when they are running standalone (192.168 network) and when they are running inside Docker Swarm (10.0 network)

### 2.3) configure Docker image build in Maven and create the Dockerfiles

configure pom.xml and Dockerfile to allow each service to run as a Docker image

    cd %HOME%\microservices\bookrecservice

edit pom.xml

add inside &lt;properties&gt;

    <docker.image.prefix>deors</docker.image.prefix>

add inside &lt;build&gt;&lt;plugins&gt;

    <plugin>
        <groupId>com.spotify</groupId>
        <artifactId>docker-maven-plugin</artifactId>
        <version>0.4.11</version>
        <configuration>
            <imageName>${docker.image.prefix}/${project.groupId}.${project.artifactId}</imageName>
            <dockerDirectory>src/main/docker</dockerDirectory>
            <imageTags>
                <imageTag>${project.version}</imageTag>
                <imageTag>latest</imageTag>
            </imageTags>
            <serverId>docker-hub</serverId>
            <resources>
                <resource>
                    <targetPath>/</targetPath>
                    <directory>${project.build.directory}</directory>
                    <include>${project.build.finalName}.jar</include>
                </resource>
            </resources>
        </configuration>
    </plugin>

create directory for Dockerfile

    mkdir src\main\docker

edit src\main\docker\Dockerfile

    FROM frolvlad/alpine-oraclejdk8:slim
    VOLUME /tmp
    ADD bookrecservice-0.0.1-SNAPSHOT.jar app.jar
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

repeat for the other projects (don't forget to update Jar file name in ADD command)

### 2.4) create the images

launch the swarm (one machine is enough)

    docker-machine start docker-swarm-manager-1 docker-swarm-manager-2 docker-swarm-manager-3 docker-swarm-worker-1 docker-swarm-worker-2 docker-swarm-worker-3

point docker client to one of the machines

when in Windows

    @FOR /f "tokens=*" %i IN ('docker-machine env docker-swarm-manager-1') DO @%i

when in Linux

    eval $(docker-machine env docker-swarm-manager-1)

build and push images by running this command for each project

    mvn package docker:build -DpushImage

### 2.5) run the images as services in Swarm

create an overlay network for all the services

    docker network create -d overlay microdemonet

launch configservice and check the status

    docker service create -p 8888:8888 --name configservice --network microdemonet deors/deors.demos.microservices.configservice:latest
    docker service ps configservice

launch eurekaservice and check the status

    docker service create -p 7878:7878 --name eurekaservice --network microdemonet -e "CONFIG_HOST=configservice" deors/deors.demos.microservices.eurekaservice:latest
    docker service ps eurekaservice

launch hystrixservice and check the status

    docker service create -p 7979:7979 --name hystrixservice --network microdemonet -e "CONFIG_HOST=configservice" -e "EUREKA_HOST=eurekaservice" deors/deors.demos.microservices.hystrixservice:latest
    docker service ps hystrixservice

launch turbineservice and check the status

    docker service create -p 8989:8989 --name turbineservice --network microdemonet -e "CONFIG_HOST=configservice" -e "EUREKA_HOST=eurekaservice" deors/deors.demos.microservices.turbineservice:latest
    docker service ps turbineservice

launch bookrecservice and check the status

    docker service create -p 8080:8080 --name bookrecservice --network microdemonet -e "CONFIG_HOST=configservice" -e "EUREKA_HOST=eurekaservice" deors/deors.demos.microservices.bookrecservice:latest
    docker service ps bookrecservice

launch bookrecedgeservice and check the status

    docker service create -p 8181:8181 --name bookrecedgeservice --network microdemonet -e "CONFIG_HOST=configservice" -e "EUREKA_HOST=eurekaservice" deors/deors.demos.microservices.bookrecedgeservice:latest
    docker service ps bookrecedgeservice

to quickly check whether all services are up and their configuration, use this command

    docker service ls

### 2.6) test services in Swarm

access the config service through one the actuator endpoints (it is currently unsecured)

    http://192.168.66.100:8888/env

check that config service is capable of returning the right configuration for some of the services

    http://192.168.66.100:8888/bookrecservice/default
    http://192.168.66.100:8888/eurekaservice/default

check that eureka service is up and the book recommendation service and edge service are registered

    http://192.168.66.100:7878/

check that hystrix service is up and running

    http://192.168.66.100:7979/hystrix

check that turbine service is up and running

    http://192.168.66.100:8989/

access the HAL browser on the book recommendation service

    http://192.168.66.100:8080/

access the book recommendation service

    http://192.168.66.100:8080/bookrec

access the HAL browser on the book recommendation edge service

    http://192.168.66.100:8181/

access the book edge recommendation service

    http://192.168.66.100:8181/bookrecedge

stop the book recommendation service, and access the book recommendation edge service again;
the default recommended book should be returned instead but the application keeps working

go back to hystrix dashboard and start monitoring the book recommendation edge service;
registering this URL in the dashboard (and optionally configuring the delay and page title)

     http://192.168.66.100:8181/hystrix.stream

once registered, try again to access the edge service, with and without the inner service up and running, and experiment how thresholds (number of errors in a short period of time) impact the opening and closing of the circuit between the inner and the edge service

### 2.7) scale out the book recommendation service

ask Docker to scale out the book recommendation service

    docker service scale bookrecservice=3

### 2.8) make updates and roll the changes without service downtime

make some change and deploy a rolling update;
for example change text string in BookController class stored here: src\main\java\deors\demos\microservices\BookController.java

rebuild and push the new image to the registry

    mvn package docker:build -DpushImage

deploy the change;
a label is needed to ensure the new version of 'latest' image is downloaded from registry

    docker service update --container-label-add update_cause="change" --update-delay 30s --image deors/deors.demos.microservices.bookrecservice:latest bookrecservice

check how the change is deployed

    docker service ps bookrecservice

## appendixes

### clean up the swarm

remove running services

    docker service rm configservice eurekaservice hystrixservice bookrecservice bookrecedgeservice

verify they are all removed

    docker service ls

remove all stored images (this must be done in every machine if more than one was used)

when in Windows

    for /F %f in ('docker ps -a -q') do (docker rm %f)
    for /F %f in ('docker images -q') do (docker rmi --force %f)

when in Linux

    docker rm $(docker ps -a -q)
    docker rmi --force $(docker images -q)

remove the nework inside swarm

    docker network rm microdemonet

stop the machines

    docker-machine stop docker-swarm-manager-1 docker-swarm-manager-2 docker-swarm-manager-3 docker-swarm-worker-1 docker-swarm-worker-2 docker-swarm-worker-3

and if desired, the swarm can be disposed, too

    docker-machine rm docker-swarm-manager-1 docker-swarm-manager-2 docker-swarm-manager-3 docker-swarm-worker-1 docker-swarm-worker-2 docker-swarm-worker-3

### troubleshooting

if using more than one machine in the swarm, images must be published to Docker Hub or another registry (for example a local registry) so they are accessible to all hosts in the swarm

to troubleshoot connectivity with curl in alpine-based images, install and use this way

    docker exec <container-id> apk add --update curl && curl <url>

to check wich IP addresses are active in a container

    docker exec <container-id> ifconfig -a

to check the environment variables in a container

    docker exec <container-id> env
