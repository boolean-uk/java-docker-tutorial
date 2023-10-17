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

When you run `build` on the project as well as doing the other steps it normally does it will also generate a `jar` file which you will be able to run from the command line using java. You'll find this file in the `build/libs` folder. Move this to a new location where we are going to build the docker image from (in theory it can just stay in the project but I move mine just to make things easier to follow).

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

For the next step you are going to need to create an account on Docker Hub and make a note of the user name you create there, as you are going to use it when creating your application, there will be a way of not having to push the image to Docker Hub and just run it locally but for now we're going to use this as it makes things portable. Run the following command in the folder where the `jar` file and `Dockerfile` currently live:

```bash
docker build -t yourusername/myapplication-name .
```

The `.` at the end is important as it tells docker to work in the current directory. Once this completes, you should be able to run your application by doing:

```bash
docker run -p 4000:4000 -d --name myapplication-name yourusername/myapplication-name
```

This detaches the process, tags it with the name and maps port 4000 internally in the application to port 4000 externally.

At this point we're going to leave it there for today, have a play with the commands etc.

Tomorrow we'll look at connecting to databases and how to hide your credentials when doing so.


