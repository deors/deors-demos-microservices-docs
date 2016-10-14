# deors.demos.microservices.docs
Microservices demo: project containing the step-by-step instructions to recreate the demo

1) set up the configuration store
---------------------------------

mkdir C:\code\deors.demos\microservices\deors.demos.microservices.configstore

cd C:\code\deors.demos\microservices\deors.demos.microservices.configstore

npp application.properties

	debug = true
	spring.jpa.generate-ddl = true

npp eureka-service.properties
	
    server.port = ${PORT:7878}
	eureka.client.register-with-eureka = false
	eureka.client.fetch-registry = false

npp hystrix-dashboard.properties

	server.port = ${PORT:7979}

npp bookrec-service.properties

	server.port = ${PORT:8080}
	eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/

initialise the Git repository

	git init
	git add .
	git commit -m "initial configuration"

publish it online (i.e. GitHub, replace with your own repository)

	git remote add origin https://github.com/deors/deors.demos.microservices.configstore.git
	git push origin master --force

2) set up the configuration server
----------------------------------

go to https://start.spring.io/

create project

	group
		deors.demos.microservices
	artifact
		deors.demos.microservices.configservice
	depedencies
		config server

extract zip to

	C:\code\deors.demos\microservices

cd C:\code\deors.demos\microservices\deors.demos.microservices.configservice

ensure config store location is properly set

npp src\main\resources\application.properties

	server.port = ${PORT:8888}
	spring.cloud.config.server.git.uri = ${CONFIG_VOL:https://github.com/deors/deors.demos.microservices.configstore.git}

configure config server to start automatically

npp src\main\java\deors\demos\microservices\Application.java

add class annotation

	@org.springframework.cloud.config.server.EnableConfigServer

3)
go to https://start.spring.io/
create project
	group
		deors.demos.microservices
	artifact
		deors.demos.microservices.eurekaservice
	depedencies
		eureka server
		config client
extract zip to
	C:\code\deors.demos\microservices
cd C:\code\deors.demos\microservices\deors.demos.microservices.eurekaservice
ensure config service is used by moving props to bootstrap phase
ren src\main\resources\application.properties bootstrap.properties
npp src\main\resources\bootstrap.properties
	spring.application.name = eureka-service
	spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}
configure eureka server to start automatically
npp src\main\java\deors\demos\microservices\Application.java
add class annotation
	@org.springframework.cloud.netflix.eureka.server.EnableEurekaServer

4)
go to https://start.spring.io/
create project
	group
		deors.demos.microservices
	artifact
		deors.demos.microservices.hystrixdashboard
	depedencies
		hystrix dashboard
		eureka discovery
		config client
extract zip to
	C:\code\deors.demos\microservices
cd C:\code\deors.demos\microservices\deors.demos.microservices.hystrixdashboard
ensure config service is used by moving props to bootstrap phase
ren src\main\resources\application.properties bootstrap.properties
npp src\main\resources\bootstrap.properties
	spring.application.name = hystrix-dashboard
	spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}
configure hystrix dashboard to start automatically
npp src\main\java\deors\demos\microservices\Application.java
add class annotation
	@org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard

5)
go to https://start.spring.io/
create project
	group
		deors.demos.microservices
	artifact
		deors.demos.microservices.bookrecservice
	depedencies
		eureka discovery
		config client
		jpa
		rest repositories
		rest repositories hal browser
		hateoas
		web
		h2
extract zip to
	C:\code\deors.demos\microservices
cd C:\code\deors.demos\microservices\deors.demos.microservices.bookrecservice
ensure config service is used by moving props to bootstrap phase
ren src\main\resources\application.properties bootstrap.properties
npp src\main\resources\bootstrap.properties
	spring.application.name = bookrec-service
	spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}
configure service to be discoverable
npp src\main\java\deors\demos\microservices\Application.java
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
generate bean constructors and methods
create BookRepository (in IDE)
	@RepositoryRestResource
	public interface BookRepository extends CrudRepository<Book, Long> {
	    @Query("select b from Book b order by RAND()")
	    List<Book> getBooksRandomOrder();
	}
create BookController (in IDE)
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

6)
configure pom.xml and Dockerfile to allow each service to run as a Docker image
cd C:\code\deors.demos\microservices\deors.demos.microservices.bookrecservice
npp pom.xml
add inside <properties>
	        <docker.image.prefix>deors</docker.image.prefix>
add inside <build><plugins>
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
mkdir src\main\docker
npp src\main\docker\Dockerfile
	FROM frolvlad/alpine-oraclejdk8:slim
	VOLUME /tmp
	ADD deors.demos.microservices.bookrecservice-0.0.1-SNAPSHOT.jar app.jar
	RUN sh -c 'touch /app.jar'
	ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
repeat for the other projects (don't forget to update Jar file in ADD command) or copy content from solution folder (with bc)

7)
launch swarm
	docker-machine start docker-swarm-manager-1
	docker-env docker-swarm-manager-1
build and push images, run this command for each project
	mvn package docker:build -DpushImage

8)
run images as containers
	docker run -p 8888:8888 --name config-service -t deors/deors.demos.microservices.configservice:latest
	docker run -p 7878:7878 -e "CONFIG_HOST=config-service" --name eureka-service -t deors/deors.demos.microservices.eurekaservice:latest
	docker run -p 7979:7979 -e "CONFIG_HOST=config-service" --name hystrix-dashboard -t deors/deors.demos.microservices.hystrixsdashboard:latest
	docker run -p 8080:8080 -e "CONFIG_HOST=config-service" -e "EUREKA_HOST=eureka-service" --name bookrec-service -t deors/deors.demos.microservices.bookrecservice:latest

9)
run images as services
	docker network create -d overlay microdemo-network
	docker service create -p 8888:8888 --name config-service --network microdemo-network deors/deors.demos.microservices.configservice:latest
	docker service ps config-service
	docker service create -p 7878:7878 -e "CONFIG_HOST=config-service" --name eureka-service --network microdemo-network deors/deors.demos.microservices.eurekaservice:latest
	docker service ps eureka-service
	docker service create -p 7979:7979 -e "CONFIG_HOST=config-service" --name hystrix-dashboard --network microdemo-network deors/deors.demos.microservices.hystrixdashboard:latest
	docker service ps hystrix-dashboard
	docker service create -p 8080:8080 -e "CONFIG_HOST=config-service" -e "EUREKA_HOST=eureka-service" --name bookrec-service --network microdemo-network deors/deors.demos.microservices.bookrecservice:latest
	docker service ps bookrec-service

10)
review state
	docker service ls
	docker service ps config-service
	docker service ps eureka-service
	docker service ps hystrix-dashboard
	docker service ps bookrec-service

11)
scale
	docker service scale bookrec-service=3

12)
make some change and deploy a rolling update
for example change text string in BookController
	mvn package docker:build -DpushImage
deploy the change; a label is needed to ensure the new version of 'latest' image is downloaded from registry
	docker service update --container-label-add update_cause="change" --update-delay 30s --image deors/deors.demos.microservices.bookrecservice:latest bookrec-service
	docker service ps bookrec-service

13)
remove running services
	docker service rm config-service
	docker service rm eureka-service
	docker service rm hystrix-dashboard
	docker service rm bookrec-service
	docker service ls
remove all stored images
	docker rm $(docker ps -a -q)
	docker rmi --force $(docker images -q)

14)
stop machine
	docker-machine stop docker-swarm-manager-1


TROUBLESHOOTING:
	- if using more than one machine in the swarm, images must be published to Docker Hub so they are accessible to all hosts in the swarm
	- if needed to troubleshoot connectivity with curl in alpine, install and use:
		docker exec <id> apk add --update curl && curl <url>
