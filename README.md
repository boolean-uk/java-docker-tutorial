# Java Docker Tutorial

## Learning Objectives
- Create a simple docker container
- Run the container and interact with it
- Make a more complex container
- Interact with that container
- Upload containers to Docker Hub
- Containerise a Spring Application, run it and interact with it

## Hello World Docker Style

To test whether your Docker installation succeeded you can just run the following command in bash:

```bash
docker run hello-world
```

It will check for a local copy of the `hello-world` docker image, if it doesn't find one it will download in and then this will output the following message if it was successful:

```bash
Hello from Docker!
This message shows that your installation appears to be working correctly.

```

Followed by some further information/instructions.

On Windows we can't just run gitbash or the Windows Subsystem for Linux command prompt straight away. 
1. We need to open the Docker Desktop app first.
2. Then click on the cog at the top right to open the `Settings` menu.
3. Go to the `Resources` section from the left hand menu.
4. Select `WSL Integration` from the sub-menu that appears and select Ubuntu and any other WSL Linux versions you want to be able to run docker from.
5. Then open your WSL command prompt and type `docker --version` to make sure it has installed.
6. You should now be able to run the `docker run hello-world` example from above.

We're going to do the rest of this using exercise using the command line version of Docker, but you should be able to replicate most of the steps in Docker Desktop too (you can try this out after we finish the run through).

## Busybox

If you've ever done any work with embedded systems or trying to get command line tools to run on Android you've probably come across `Busybox` it is a tool which when installed replicates a lot of common command line tools for bash inside a single binary. We can create/download a docker container which has Busybox installed and ready to go really easily.

Run the following command:

```bash
docker pull busybox
```

This will `pull` a docker image which contains `Busybox` from the `Docker Registry` and save it onto our machine.

We can run it by typing:

```bash
docker run busybox
```

But this won't have any visible effect.

Try 

```bash
docker run busybox echo "Well hello there."
```

and you should see the message output to the shell.

Running 

```bash
docker ps
```

will show you any running docker images, 

```bash
docker ps -a
```

will show you previously running ones and


```bash
docker images
```

will show you the base images currently downloaded onto your machine.

To start an interactive `busybox` session we can do:

```bash
docker run -it busybox sh
```

and then type various bash commands in the resulting prompt to experiment. When you're done type `exit` to exit the session.

We can remove the previous sessions by running 

```bash
docker container prune
```

## Using Somebody Else's Image

We can use existing images from the Docker Registry provided we know their name. Here's a static-site one that features in a reasonably popular tutorial about docker, we can download and run it all in a single step like this:

```bash
docker run --rm -it prakhar1989/static-site
```

The `--rm` option will automatically remove the session when we finish running it. It may take a little while to download the first time you run it, but once it has been downloaded it will check if a newer version exists and just run the local copy from then onwards. If successful you'll see:

```bash
Nginx is running...
```

in the terminal but no indication of how to reach the site that is being served. To actually get the site accessible to us we need to tell docker which port to use. Hit `Ctrl-C` to stop the container running. Then do the following to run the container again:

```bash 
docker run -d -P --name static-site prakhar1989/static-site
```

The `-d` option detaches the terminal and the container from each other. `-P` assigns random numbers to any available ports and `--name` gives the container a human readable label we can use to access information about it. We can find out the ports that were assigned by doing:

```bash
docker port static-site
```

We can then go to `localhost` followed by the port number to access the site that's being served. (The one that port 80 maps to will probably work, the one that port 443 maps to probably won't as there is no certificate associated with it.)

Run the following to stop it running:

```bash
docker stop static-site
```

You can also specify which port to use when running the application by doing:

```bash
docker run -d -p 4000:80 --name static-site prakhar1989/static-site
```

But to access the resulting site I had to connect to `0.0.0.0:4000` rather than `localhost:4000` I am not sure why.

## Creating Our Own Docker Image

It's fairly straightforward to create a new Docker Image which we can then use to run our applications. If we wanted to have a Linux image based on an old version of Ubuntu (18.04 for instance) then we could do

```bash 
docker pull ubuntu:18.04
```

To download the base ubuntu 18.04 image. We can do similar with various other base images. To create an image that we can use with one of our Java Applications we'll need to make a slight modification to it so that it produces a jar file that we can then run inside our docker container. 

Open `build.gradle` in the Project you want to use (I did this with one of the self contained Spring ones) and add the following at the bottom:

```groovy
jar {
	duplicatesStrategy = DuplicatesStrategy.EXCLUDE
	manifest {
		attributes 'Main-Class': 'com.booleanuk.api.Main'
	}

	from {
		configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) }
	}
}
```

When you run `build` on the project as well as doing the other steps it normally does it will also generate a `jar` file which you will be able to run from the command line using java. You'll find this file in the `build/libs` folder. Move this to a new location where we are going to build the docker image from (in theory it can just stay in the project but I move mine just to make things easier to follow). If you only get a `jar` file with the word `plain` in the filename then this won't work and you may need to open a terminal window and then enter the command:

```bash
./gradlew build
```

into it, which will cause your `jar` file to be generated.

You can test that the `jar` file works by opening a bash/gitbash terminal in the folder where you've copied it and running the command:

`java -jar requests-0.0.1-SNAPSHOT.jar`

Once you're happy in the new location, create a new file called `Dockerfile` and add the following contents to it

```
FROM eclipse-temurin:18

LABEL maintainer="david.john.ames@gmail.com"

WORKDIR /app

COPY requests-0.0.1-SNAPSHOT.jar /app/requests-0.0.1-SNAPSHOT.jar

EXPOSE 4000

ENTRYPOINT ["java", "-jar", "requests-0.0.1-SNAPSHOT.jar"]
```

For the next step you are going to need to create an account on Docker Hub and make a note of the user name you create there, as you are going to use it when creating your application, if you choose to you will then be able to `push` your docker image up to Docker Hub to share with people. **Make sure you do not do this with database credentials or other secrets included!!!!!** Run the following command in the folder where the `jar` file and `Dockerfile` currently live:

```bash
docker build -t yourusername/myapplication-name .
```

The `.` at the end is important as it tells docker to work in the current directory. Once this completes, you should be able to run your application by doing:

```bash
docker run -p 4000:4000 -d --name myapplication-name yourusername/myapplication-name
```

This detaches the process, tags it with the name and maps port 4000 internally in the application to port 4000 externally.

## Connecting to an ElephantSQL Database

You can follow the same procedure as above to generate the `jar` file and then just create a new Docker Image and run it. It will seamlessly connect to the remote database using the credentials you supplied in the `application.yml` file. If you share this onto Docker Hub then other people will be able to run your image and connect to your database using it, and the credentials are also stored in plain text inside the `jar` file. Do not do this it is a very bad idea!!!!

## Connecting to a Local Database also in a Docker Image

We are going to make it so that our application running in its container connects to a Postgres database that runs in its own container, with both containers on the same machine. If we wanted to have our Postgres container on a separate machine, then as long as we started it correctly on that machine and connected up the correct port then we should be able to just connect to it as though it was running directly on the remote machine and listening on the given port (ie port 5432 the default Postgres port).

When we run our Spring Application in a container it sees the `localhost` as being inside the machine, so if we wanted to connect to another service running on the same host machine then we would need to know the IP address of that host in order to connect to it, but if the service is itself in a container then we can use a "trick" to connect them both together using docker itself. What we basically do is create a docker `network` that will allow us to connect together containers without worrying about what IP address they're running on, if  we needed to there are ways to lock down this functionality to prevent other users from gaining access to it or intercepting traffic on it.

Let's start by creating a new docker network which we'll use to let our Spring App container and our Database container talk to each other. In the terminal type:

```bash
docker network create daves-music-net
```

The `daves-music-net` part is the name of the docker network that has just been created. Run:

```bash
docker network ls
```

to see a list of all of the networks currently running. If you don't remove the network it will remain on your system even after rebooting.

First we're going to download and run an instance of `postgres` in its own container. Let's start by pulling the official `postgres` image from Docker Hub using: 

```bash
docker pull postgres
```

Then we can run it, on our docker network using the following command:

```bash
docker run --name postgresdb --network daves-music-net -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DATABASE=postgres -e POSTGRES_USER=postgres -d postgres
```

Check that it's running using `docker ps`. Now we're going to build the `jar` file for our Spring Application but with the following connection details in our `application.yml` file:

```yml
server:
  port: 4000
  error:
    include-message: always
    include-binding-errors: always
    include-stacktrace: never
    include-exception: false

spring:
  datasource:
    url: jdbc:postgresql://postgresdb:5432/postgres
    username: postgres
    password: mypassword
    max-active: 3
    max-idle: 3
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
      show_sql: true
```

The database url uses the name of our postgres docker container `postgresdb`, the database name at the end of the url and the username are the ones we specified in the command we ran the database with above as is the password. To build the application we will need to make it ignore the built in tests, as otherwise the build will fail. Open the terminal in the top level of the Spring Application and run the following command which ignores any tests and generates the required `jar` file.

```bash
./gradlew build -x test
```

Copy the resulting `jar` file into the folder with a Dockerfile that has the same basic contents as last time and then build the image with:

```bash
docker build -t amos1969/daves-music-box .
```

or your equivalent. If that succeeds then you should be able to run it in the same docker network as the database with: 

```bash
docker run --network daves-music-net --name daves-music-box -p 4000:4000 -d amos1969/daves-music-box
```

You should then be able to fire up Insomnia and interact with your endpoints using that. Unless you've created records in the database tables there won't be anything in there. You should be able to stop and start the containers without too much problem. If you delete the instance of the database one and then rerun it using the same commands as previously the data will not persist between runs. You can overcome this by connecting the database to a `volume` to persist data but we aren't going to go into that here.

Until you prune the container instance, you can always restart it using `docker start container-instance-name` so if I had stopped the Spring Applicatation container I could restart it using `docker start daves-music-box` and it would restart with the same settings as we previously gave it. If I called prune on any stopped containers I would need to redo the whole `run` command as shown above and any data in the database would  be lost if the database container was the one which was pruned and then rerun.

## Using docker-compose to Simplify Things

Although we can now start multiple containers and make them talk to each other, there is a certain amount of prior knowledge needed in order to do so, we can instead use a command called `docker-compose` to tie everything together and make them all run simultaneously from a single command.

We're going to use a Docker Compose file to deal with all of the database connection shenanigans this time, so delete the database part (everythiong from the word `spring` down) from the `application.yml` file and then rebuild the spring app, using `./gradlew build -x test`. Copy the `jar` file across and build a new image from it (I'm calling mine `amos1969/music-box`).

Now create a new file called `docker-compose.yml` somewhere, I put this in the same folder where my `jar` file and `Dockerfile` are but just to keep things together, there is no need for them to be in the same place.

Inside the `docker-compose.yml` file we are going to detail the two images we want to talk to each other as well as the connection details we previously had inside `application.yml`. Each image is known as a `service` and we need to specify various aspects of how the two services will communicate with each other. Here's a working one:

```yml
services:
  app:
    image: 'amos1969/music-box:latest'
    container_name: app
    depends_on:
      - db
    ports:
      - '4000:4000'
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mypostgresuser
      - SPRING_DATASOURCE_USERNAME=mypostgresuser
      - SPRING_DATASOURCE_PASSWORD=mypostgrespassword
      - SPRING_JPA_HIBERNATE_DDL_AUTO=update

  db:
    image: 'postgres:latest'
    container_name: db
    environment:
      - POSTGRES_USER=mypostgresuser
      - POSTGRES_DATABASE=mypostgresuser
      - POSTGRES_PASSWORD=mypostgrespassword
```

I'm unclear whether the service name and the container name need to match. It appears that the database name at the end of the URL and in the database details needs to match the user name for Postgres. The `depends_on` setting tells docker to spin up the database one first and then allow the application to talk to it. In the background this will create a default docker network for the containers to talk to each other over.

Once everything;s set up, use:

```bash
docker-compose up -d
```

to start it running, and detach the process from the terminal.

Going to http://localhost:4000 on my local machine allows me to interact with the endpoints as normal. If you had a React frontend that was also in a container then you would talk to that and it in turn would talk to the API.

Things you need to investigate further are:
- How to secure this sort of set up?
- How a React container will know which port of the API to talk to?
- How to use volumes to persist the data in the database?


