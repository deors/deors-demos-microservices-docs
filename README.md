# deors.demos.microservices.docs
Microservices demo: repository containing the step-by-step instructions to recreate the demo

0) prerequisites
----------------

NOTE: the following instructions are created on a Windows machine - some commmands may need
slight adjustments when working on Linux/OSX

this demo assumes that an existing docker swarm is available

in my case, I have 6 machines on VirtualBox named:

    docker-machine start docker-swarm-manager-1
    docker-machine start docker-swarm-manager-2
    docker-machine start docker-swarm-manager-3
    docker-machine start docker-swarm-worker-1
    docker-machine start docker-swarm-worker-2
    docker-machine start docker-swarm-worker-3

those machines are deployed in its own network:

    192.168.88.1/24

being the first IP in DHCP pool:

    192.168.88.100

1) set up the configuration store
---------------------------------

create and change to a directory for the project

    mkdir %HOME%\microservices\deors.demos.microservices.configstore
    cd %HOME%\microservices\deors.demos.microservices.configstore

create file application.properties

    debug = true
    spring.jpa.generate-ddl = true

create file eureka-service.properties
    
    server.port = ${PORT:7878}
    eureka.client.register-with-eureka = false
    eureka.client.fetch-registry = false

create file hystrix-dashboard.properties

    server.port = ${PORT:7979}

create file bookrec-service.properties

    server.port = ${PORT:8080}
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/

initialise the Git repository

    git init
    git add .
    git commit -m "initial configuration"

publish it online (i.e. GitHub, replace with your own repository)

    git remote add origin https://github.com/deors/deors.demos.microservices.configstore.git
    git push origin master

2) set up the configuration server
----------------------------------

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: deors.demos.microservices.configservice
    depedencies: config server

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\deors.demos.microservices.configservice

ensure config store location is properly set;
edit src\main\resources\application.properties

    server.port = ${PORT:8888}
    spring.cloud.config.server.git.uri = ${CONFIG_VOL:https://github.com/deors/deors.demos.microservices.configstore.git}

configure config server to start automatically;
edit src\main\java\deors\demos\microservices\Application.java

add class annotation

    @org.springframework.cloud.config.server.EnableConfigServer

3) set up the Eureka server
---------------------------

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: deors.demos.microservices.eurekaservice
    depedencies:
        eureka server
        config client

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\deors.demos.microservices.eurekaservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = eureka-service
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure eureka server to start automatically;
edit src\main\java\deors\demos\microservices\Application.java

add class annotation

    @org.springframework.cloud.netflix.eureka.server.EnableEurekaServer

4) set up the Hystrix dashboard
-------------------------------

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: deors.demos.microservices.hystrixdashboard
    depedencies:
        hystrix dashboard
        eureka discovery
        config client

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\deors.demos.microservices.hystrixdashboard

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = hystrix-dashboard
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure hystrix dashboard to start automatically;
edit src\main\java\deors\demos\microservices\Application.java

add class annotation

    @org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard

5) set up book recommendation service
-------------------------------------

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: deors.demos.microservices.bookrecservice
    depedencies:
        eureka discovery
        config client
        jpa
        rest repositories
        rest repositories hal browser
        hateoas
        web
        h2

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\deors.demos.microservices.bookrecservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = bookrec-service
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure service to be discoverable;
edit src\main\java\deors\demos\microservices\Application.java

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

6) dockerize the services
-------------------------

configure pom.xml and Dockerfile to allow each service to run as a Docker image

    cd %HOME%\microservices\deors.demos.microservices.bookrecservice

edit pom.xml

add inside &lt;properties&gt;

    <docker.image.prefix>deors</docker.image.prefix>

add inside &lt;build&gt;&lt;plugins&gt;

    <plugin>
        <groupId>com.spotify</groupId>
        <artifactId>docker-maven-plugin</artifactId>
        <version>0.4.11</version>
        <configuration>
            <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
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
    ADD deors.demos.microservices.bookrecservice-0.0.1-SNAPSHOT.jar app.jar
    RUN sh -c 'touch /app.jar'
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

repeat for the other projects (don't forget to update Jar file in ADD command)

7) creating the images
----------------------

launch swarm (one machine is enough)

    docker-machine start docker-swarm-manager-1
    docker-machine start docker-swarm-manager-2
    docker-machine start docker-swarm-manager-3
    docker-machine start docker-swarm-worker-1
    docker-machine start docker-swarm-worker-2
    docker-machine start docker-swarm-worker-3

point docker client to one of the machines

    docker-env docker-swarm-manager-1

build and push images by running this command for each project

    mvn package docker:build -DpushImage

8) running the images as services
---------------------------------

create an overlay network for all the services

    docker network create -d overlay microdemo-network

launch config-service and check the status

    docker service create -p 8888:8888 --name config-service --network microdemo-network deors/deors.demos.microservices.configservice:latest
    docker service ps config-service

launch eureka-service and check the status

    docker service create -p 7878:7878 -e "CONFIG_HOST=config-service" --name eureka-service --network microdemo-network deors/deors.demos.microservices.eurekaservice:latest
    docker service ps eureka-service

launch hystrix-dashboard and check the status

    docker service create -p 7979:7979 -e "CONFIG_HOST=config-service" --name hystrix-dashboard --network microdemo-network deors/deors.demos.microservices.hystrixdashboard:latest
    docker service ps hystrix-dashboard

launch bookrec-service and check the status

    docker service create -p 8080:8080 -e "CONFIG_HOST=config-service" -e "EUREKA_HOST=eureka-service" --name bookrec-service --network microdemo-network deors/deors.demos.microservices.bookrecservice:latest
    docker service ps bookrec-service

9) test services
----------------

access config service

    http://192.168.88.100:8888/env

check config service is capable of returning the right configuration for some of the services

    http://192.168.88.100:8888/bookrec-service/default
    http://192.168.88.100:8888/eureka-service/default

check eureka service is up and book recommendation service is registered

    http://192.168.88.100:7878/

access the HAL browser on the book recommendation service    

    http://192.168.88.100:8080/

access the book recommendation service

    http://192.168.88.100:8080/bookrec

10) scale the book recommendation service
-----------------------------------------

ask Docker to scale the service

    docker service scale bookrec-service=3

11) make updates and roll the changes without service downtime
--------------------------------------------------------------

make some change and deploy a rolling update;
for example change text string in BookController class stored here: src\main\java\deors\demos\microservices\BookController.java

rebuild and push the new image to the registry

    mvn package docker:build -DpushImage

deploy the change;
a label is needed to ensure the new version of 'latest' image is downloaded from registry

    docker service update --container-label-add update_cause="change" --update-delay 30s --image deors/deors.demos.microservices.bookrecservice:latest bookrec-service

check how the change is deployed

    docker service ps bookrec-service

12) clean up
------------

remove running services

    docker service rm config-service
    docker service rm eureka-service
    docker service rm hystrix-dashboard
    docker service rm bookrec-service

verify they are all removed

    docker service ls

remove all stored images (this must be done in every machine if more than one was used)

if in windows

    for /F %f in ('docker ps -a -q') do (docker rm %f)
    for /F %f in ('docker images -q') do (docker rmi --force %f)

if in linux

    docker rm $(docker ps -a -q)
    docker rmi --force $(docker images -q)

stop the machines

    docker-machine stop docker-swarm-worker-3
    docker-machine stop docker-swarm-worker-2
    docker-machine stop docker-swarm-worker-1
    docker-machine stop docker-swarm-manager-3
    docker-machine stop docker-swarm-manager-2
    docker-machine stop docker-swarm-manager-1


TROUBLESHOOTING:
----------------

if using more than one machine in the swarm, images must be published to Docker Hub so they are accessible to all hosts in the swarm

if needed to troubleshoot connectivity with curl in alpine-based images, install and use this way

    docker exec <id> apk add --update curl && curl <url>
