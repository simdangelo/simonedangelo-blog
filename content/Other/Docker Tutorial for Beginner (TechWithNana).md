---
date: 2024-03-16
tags:
  - docker
modified: 2024-06-18T21:14:10+02:00
---

In this tutorial, I will share the insights and tips I gathered from a video on YouTube by "**TechWorld with Nana**" channel (link here: [video tutorial here]( https://www.youtube.com/watch?v=3c-iBn73dDE&ab_channel=TechWorldwithNana)). These notes have been organized to follow the video step by step.
# Main Docker Commands
>[!tip]
>**Container**: running environment for **IMAGE**.

![Untitled](Other/attachments/Untitled%206.png)
As you can see from the picture above, the application image (the application could be Postgres, Redis, etc…) needs a File System where it can save the log files or where you can store some configuration files and it needs the environmental configuration, like environmental variables and so on. All this stuff are provided by the container and container also has a port that is binded to it, which makes it possibile to talk to the application which is running inside the container.

Note that the File System is virtual in container. So to container has its own abstraction of an operating system, including the File System and the Environment, which is different from the File System and the Environment of the host machine.

To see the difference from Image and Container, go to DockerHub. Note that all the artifacts that are in Docker are Images, so we’re not talking about container. If you look for Redis, you’ll find the following command:
```powershell
docker pull redis
```
You’ll see that different layers of the image are downloading:
![Untitled](Other/attachments/Untitled%207.png)

Once the download is complete, you can check all the existing images using:
```powershell
docker images
```

![Untitled](Other/attachments/Untitled%208.png)

Another important aspects of Images is that they take TAG or VERSION. If you go back to the DockerHub, each image that you look up will have many different versions:
![Untitled](Other/attachments/Untitled%209.png)

The `latest` is the one that you get when you don’t specify the version.

So far, we only worked with Images, there is no container involved.

Now let’s say I need Redis running so that my application can connect to it, so I’ll have to create a Container of that Redis Image, that will make possibile to connect to the Redis Application. We can do it by:
```powershell
docker run redis
```

![Untitled](Other/attachments/Untitled%2010.png)

This actually starts the Image in the Container. As we said before, a Container is a running environment of an Image.

If you open another terminal window, you can see all the **running Containers** with:
```powershell
docker ps
```
![Untitled](Other/attachments/Untitled%2011.png)

So `docker run redis` command will start the Redis container in the terminal in an **Attached Mode**, meaning that the terminal is running forever after entering some command. This indicates the server is running. If I were to terminate this with `control+c`, the Redis application stops and the container will be stopped as well. Indeed, if you hit again `docker ps`, you will find that no container is running.

So there is an option for `docker run` that it able makes it possible to run the container in **Detached Mode** that is `-d`:
```powershell
docker run -d redis
```
![Untitled](Other/attachments/Untitled%2012.png)

In this case I will just get the ID of the container, that you can see by `docker ps`.

If you want to **restart the container** because for example some applications crash inside, you need the CONTAINER ID and hit:
```powershell
docker stop acf10d912cba
```
(it’s not necessary to write the whole ID, just the partial string you’ll find with `docker ps`.

If you want to start it again after stopped it:
```powershell
docker start acf10d912cba
```
This is something that you can do after a work day. Stop it when your work day is finished and start it the next day.

You can list all the running and stopped container with:
```powershell
docker ps -a
```
![Untitled](Other/attachments/Untitled%2013.png)

Let’s say you have two parallel applications that both use Redis, but in different version; so you would need two different containers with different image versions running on your laptop.
First of all, notice that when you run Redis container for the first time, you got the latest version and you can read that in the log appeared after `docker run redis` command:
![Untitled](Other/attachments/Untitled%2014.png)

In this case the latest version is `7.2.3`.
If you go to DockerHub, you can find another version that you are looking for, for example the `4.0`. We can **pulls the image and starts the container** with a unique command by:
```powershell
docker run redis:4.0
```
![Untitled](Other/attachments/Untitled%2015.png)

Now, if you hit docker ps, you’ll get two Redis running containers:
![Untitled](Other/attachments/Untitled%2016.png)

How do you actually use any container that you just started?
Look at the PORTS specs: that specify on which port the container is listening to the incoming request. So, both containers open the same port, which is what was specified in the image. So, how do we not have conflicts while both are running on the same port? Let’s see how this works.
You can have multiple containers running simultaneously on your host, which is your laptop, PC, or whatever you are working on, and your laptop has some **ports available** that you can open for certain applications. This works by creating a so-called **binding between a port that your laptop has and the container**.
![Untitled](Other/attachments/Untitled%2017.png)

In this picture you may see some bindings. You will have conflict if you open two 5000 ports on your host because you’ll get a message saying that that port is already in use. However you can have two containers, as you can see in the second and third containers in the picture above, that are both listening on port 3000, which is absolutely ok as long as you’re bind them to two different ports from your laptop.
Once the binding is done, you would have
![Untitled](Other/attachments/Untitled%2018.png)

With this configuration, the host will know how to forward requests to the container using the port binding.
Going back to this picture
![Untitled](Other/attachments/Untitled%2016.png)

you may see that containers have their port (`6379`). However, we haven’t made any binding between our laptop’s ports and the container port. And, because of that, the container is basically **unreachable** by any application, so I won’t be able to use it.
So, the way we actually do that is by specifying the binding of the ports during the run command with the keyword `-p`:
```powershell
docker run -p6000:6379 redis
```
Now, type `docker ps`:
![Untitled](Other/attachments/Untitled%2019.png)

You can see the binding under the PORTS spec. 
Now hit `control+c` to stop the application and start again it in detached mode:
```powershell
docker run -p6000:6379 -d redis
```
And now let’s start the second container:
```powershell
docker run -p6001:6379 -d redis:4.0
```
and check the running containers:
![Untitled](Other/attachments/Untitled%2020.png)

Now you have two different Redis versions running both of them bounding to different ports on my laptop (`6000` and `60001`) and the containers themselves listening to rquest on the same port (`6379`).
# Debugging a container
So far, we have seen:
* `docker pull`: pulls the image from the repository to local environment
* `docker run`: combines pool and start (pulls the image if it’s not locally available and then start it)
* `docker start` and `docker stop`: makes it possibile to restart the container if you made some changes and you want to create a new version.
* `docker run -d`: detached mode.
* `docker run -p`: allows you to bind port of your host to the container.
* `docker ps`: lists all running containers.
* `docker ps -a`: lists all the containers, no matter if they’re running or not.
* `docker images`: lists all the images that you have locally.

Now we’ll see commands for **troubleshooting**: if something goes wrong in the container, you want to see the logs of the container, or you want to actually get inside the container, get the terminal and execute some commands on it.
Now you have 2 running Redis containers. Imagine that your application cannot connect to Redis and you don’t know what’s happening. You would want to see what logs Redis container id producing. You can do with `docker logs` specifying the container ID:
```powershell
docker logs e3b68c4ce6f5
```
![Untitled](Other/attachments/Untitled%2021.png)

Note1: that you can use the container name (flamboyant_meitner) instead of the container ID.
Note2: container name are given automatically. You can give a specific name to container. So let’s stop the Redis 4.0 container with docker stop and then use the `—-name` option to create a new container: 
```powershell
docker run -d -p6001:6379 --name redis-older redis:4.0
```

Let’s do the same for Redis container with the latest version:
```docker
docker stop 87eaa6b4135b
docker run -d -p6000:6379 --name redis-latest redis
```
![Untitled](Other/attachments/Untitled%2022.png)

Another useful command in debugging is `docker exec`, with which you can get the **terminal of a running container**:
```powershell
docker exec -it b957257fb491 /bin/bash
```
Note1: `-it` stands for **interactive terminal**.
Note2. You can also use the NAME instead of the ID.
![Untitled](Other/attachments/Untitled%2023.png)

You may see that the cursor changed and this means you are inside the container.
With `pwd` command you may see that you are inside the path `/data`. So let’s go inside the home directory with `cd/`, then hit `ls`:
![Untitled](Other/attachments/Untitled%2024.png)

This is the virtual File System inside the container. Now you can navigate wherever you want if you want for example to debug something. You can also print the environmental variables with `env` command if you want to see if they’re set correctly:
![Untitled](Other/attachments/Untitled%2025.png)

This is very useful when you are running your own application that you wrote in a container and you have some complex configuration or setup that you want to validate that everything is correctly set.

To exit from the container just type `exit`.

Since most of container images are based on some lightweight Linux distributions, you will not have much of the Linux commands or applications installed here. For example, you wouldn’t have `curl` or some other stuff. So you were a little bit limited in that sense.

Let’s review two commands to explain better the differences: `docker run` and `docker start`.
- `docker run`: you create a new container from an image. So `docker run` command will take an image with a specific version (or `latest`).
- `docker start`: you are not working with images, but rather with containers. Once you create a new container, you can start it after stopped by specifying the ID of that specific container. And when we start that container after stopped it, it will retain all the attributes (NAMES, PORTS, etc) that we defined when creating the container with `docker run`.
# Demo Project Overview - Docker in Practice
Demo project.

**Step1.** You’re developing a JavaScript application in your laptop. This application uses MongoDB database and, instead of installing it on your laptop, you download a docker container from the DockerHub. With with configuration, suppose you develop the first version of this application locally, but now you want to test it or you want to deploy it on the development environment where a tester in your team is going to test it.
**Step2**. So you commit your JavaScript application in Git or in some other version control system that will trigger a continuous integration with, for example, Jenkins.
**Step3**. Jenkins will produce Artifacts from you application. So first you will build your JavaScript application and then create a Docker image out of that JavaScript artifacts. So what happens to this Docker Image once it gets created by Jenkins?
**Step4**. It gets pushed to a Private Docker Repository. Usually, in your company, you will have a private repository because you don’t want other people to have access to your images.
**Step5**. The next step could be configured on Jenkins or some other scripts or tools, that Docker Image has to be deployed on a Development Server. So you have a Development Server that pulls both:
- the image from the private docker repository (the JavaScript application image);
- the MongoDB that your JavaScript application depends on from DockerHub.
Now you have two containers, one is your custom container the other one is a publicly available MongoDB container, running on Dev Server, and, after configuration, they talk and communicate to each other and run as an App. So, now if a Tester logs into a Dev Server, they will be able to test the application.
![Untitled](Other/attachments/Untitled%2026.png)

Let’s see it in practice.
# Developing with Containers

![Untitled](Other/attachments/Untitled%2027.png)
What we’ll create:
- simple UI backend application using JavaScript, very simple HTML structure and NodeJS in the backend:
- in order to integrate all of this in the database, we’re gonna use a docker container of MongoDB database and, to makes it working with MongoDB much easier so we don’ have to execute commands in the terminal, we’re going to deploy a docker container of a Mongo UI, which is called MongoExpress, where we can see the database structure and all the update that out application is making in the database.

The result is below:
![Untitled](Other/attachments/Untitled%2028.png)
Let’s see both of the 2 steps.
### JavaScript application
Now, let’s start from the first part. This is a very simple User Profile page, where you can change personal information:
![Untitled](Other/attachments/Untitled%2029.png)

With Edit Profile button you can modify personal info, but if you refresh the page the changes will be lost because it’s just JavaScript and NodeJS, and there is no persistence component in the application. So, in order to have it, you need to integrate the application with a database. We’ll do that by pulling one of the DB and attaching it or connecting it to the application. In this case we’re going to go with the MongoDB application and, in addition to it, we’re going to deploy a MongoDB UI, which is its own container, that is called MongoExpress.
### MongoDB Docker Container
Go to DockerHub and find the MongoDB image and follow the instruction to pull the image. In particular:
```powershell
docker pull mongo
```

Do the same with MongoExpress:
```powershell
docker pull mongo-express
```

Run both containers in order to make the MongoDB DB available for the application and also to connect MongoExpress with MongoDB container.

Let’s make first the connection between MongoDB and MongoExpress. In order to do that, we have to understand another Docker concept, **Docker Network**. How it works?

Docker creates **Isolated Docker Network**, where the containers are running in. So when I deploy two containers in the same Docker Network (in this case MongoDB and MongoExpress), they can talk to each other using just the container name without [localhost](http://localhost) and the port number because they are in the same network.

The applications that run outside of Docker, like out NodeJS which just runs from Node server, is going to connect to them from outside or from the host using [localhost](http://localhost) and the port number.
![Untitled](Other/attachments/Untitled%2030.png)

So later, when we package out application in its own docker image, what we’re going to have is again a Docker Network with MongoDB container, MongoExpress container, and we’re going to have a NodeJS application that we wrote, including the index.html and JavaScript for frontend, in its own Docker Container and it’s going to connect to the MongoDB database.
![Untitled](Other/attachments/Untitled%2031.png)

And the browser, which is running on the host, but outside the Docker Network is going to connect to our JS application again using host name and post number.
If you hint:

```powershell
docker network ls
```

![Untitled](Other/attachments/Untitled%2032.png)

We’re going to create its own network for MongoDB and MongoExpress and we’re going to call it **Mongo Network:**

```powershell
docker network create mongo-network
```

![Untitled](Other/attachments/Untitled%2033.png)

In order to make our MongoDB container and the MongoExpress container run in this mongo-network we have to provide this network option when we run the container in the `docker run` command. Note that in this run command we have to specify some **environmental variables** (you can find information in Mongo Image page in DockerHub site).

![Untitled](Other/attachments/Untitled%2034.png)

```powershell
docker run -d \                                                             
-p 27017:27017 \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD=password \
--name mongodb \
--net mongo-network \
mongo
```

Now, let’s start MongoExpress and we want it to connect to the running MongoDB container on startup. Let’s see in the DockerHub the environmental variables:

![Untitled](Other/attachments/Untitled%2035.png)

ME_CONFIG_MONGODB_ADMINUSERNAME and ME_CONFIG_MONGODB_ADMINPASSWORD are the ones that we overwrite when running the MongoDB container and we’ll use them. The ME_CONFIG_MONGODB_PORT is by default the correct one (27017 already used when running MongoDB). ME_CONFIG_MONGODB_SEVER is important because it refers to the container name that MongoExpress uses to connect to the Docker and, because they’re running in the same network and only because of that, this configuration will work. If I hadn’t specify the network then I could specify the name of the container, but it wouldn’t work.

```powershell
docker run -d \
-p 8081:8081 \
-e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
-e ME_CONFIG_MONGODB_ADMINPASSWORD=password \
--net mongo-network --name mongo-express \
-e ME_CONFIG_MONGODB_SERVER=mongodb \
mongo-express
```

If you navigate to [http://localhost:8081/](http://localhost:8081/) you’ll see (username and pass are shown after the run command):

![Untitled](Other/attachments/Untitled%2036.png)

Let’s create a new DB called `user-account` and now we can actually use it to connect to this DB from NodeJS. Let’s see how that works.

### Connect Node Server with MongoDB container

We’re going to see the following part:

![Untitled](Other/attachments/Untitled%2037.png)

*(from this point on I need the code from the video creator; you can download from the url under the video description but maybe you need to install something in order to makes NodeJS working. Maybe it’s not difficult, but i skip this part at the moment. She basically just edit the profile and this changes are permanent because now we have a DB where all these actions are registered)*

# Docker Compose - Running multiple services

In the previous section we created and started two docker containers, one for MongoDB and the other one for MongoExpress and we used the following commands:

```powershell
## create docker network
docker network create mongo-network

## start mongodb
docker run -d \                                                             
-p 27017:27017 \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD=password \
--name mongodb \
--net mongo-network \
mongo

## start mongo-express
docker run -d \
-p 8081:8081 \
-e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
-e ME_CONFIG_MONGODB_ADMINPASSWORD=password \
--net mongo-network --name mongo-express \
-e ME_CONFIG_MONGODB_SERVER=mongodb \
mongo-express

```

This way of starting containers all the time is a little bit tedious and you don’t want to execute these commands all the time on the command line terminal, especially if you have a bunch of docker container to run. You want to automate it or just make it a little bit easier. There is a tool that makes running multiple docker containers with all these configurations much easier than with docker run command, and that is **Docker Compose**.

In a Docker Compose file we can take the whole commands with its configuration and map it into a file so that we have structured commands. If you have 10 docker containers that you want to run for your application and they’ll need to talk to each other, you can basically write all the run commands for each container in a structured way.

Let’s see this mapping for MongoDB container:

![Untitled](Other/attachments/Untitled%2038.png)

![Untitled](Other/attachments/Untitled%2039.png)

![Untitled](Other/attachments/Untitled%2040.png)

![Untitled](Other/attachments/Untitled%2041.png)

![Untitled](Other/attachments/Untitled%2042.png)

Let’s do the same for MongoExpress container:

![Untitled](Other/attachments/Untitled%2043.png)

![Untitled](Other/attachments/Untitled%2044.png)

![Untitled](Other/attachments/Untitled%2045.png)

![Untitled](Other/attachments/Untitled%2046.png)

So basically, **Docker Compose** is a structured way to contain very normal common Docker commands. It will be easier for us to edit the file, if you want to change some variable or change the ports or add some new option.

You may notice that the Docker Network configuration is not in the docker compose. This is because docker compose takes care of creating common Network:

![Untitled](Other/attachments/Untitled%2047.png)

### Creating the Docker Compose File

(Pay attention at the next block of code because in the slide the connection for mongo-express container was at the port 8080, but in the following will be 8081)

```yaml
version: '3'
services:
  mongodb:
    image: mongo
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
  mongo-express:
    image: mongo-express
    ports:
      - 8080:8081
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_MONGODB_SERVER=mongodb
```

Save this file as `mongo.yaml`. Note that indentation in yaml file is important.

Once you have this Docker Compose File, how to start the containers using that? Go to terminal and:

```powershell
docker-compose -f mongo.yaml up
```

Note: if you installed Docker on your laptop, it usually gets installed with **Docker Compose** package inside.

Docker Compose command takes

- an argument which is the file with `-f mongo.yaml`
- at the end specify what I want to do with this file, in this case `up` which start all the containers which are in the `mongo.yaml`.

Now check before that there are no containers running with `docker ps` and then hit the command `docker-compose -f mongo.yaml up`. We get a series of logs. Let’s examine the most important ones.

Before we talked about Docker Network and how to create our own network to run the containers inside and we said that Docker Compose takes care of it. We can see that in this logs:

![Untitled](Other/attachments/Untitled%2048.png)

The first line is the Docker Network creation; the second and third lines represents the creation of the two containers we need and the names are automatically created with prefix and suffix. If we hit `docker network ls` we can see the new Docker Network:

![Untitled](Other/attachments/Untitled%2049.png)

Note that the logs of te two containers actually mixed because we started both at the same time:

![Untitled](Other/attachments/Untitled%2050.png)

In this case, mongo-express container has to wait for mongodb container to start because it need to establish a connection. Indeed if we scroll the logs, we’ll have:

![Untitled](Other/attachments/Untitled%2051.png)

Finally, after mongodb started, mongo-express is listening for connection and he’s able to connect to it.

Of course you can see the running container with `docker ps`:

![Untitled](Other/attachments/Untitled%2052.png)

One thing to note is that the MongoExpress actually started on Port 8081 inside the container, so we’re opening the Port 8080 on our laptop that actually forward the request to container at port 8081.

So, now navigate to [localhost:8080](http://localhost:8080) and we get the same result as before:

![Untitled](Other/attachments/Untitled%2053.png)

Remember that in the previous example we created a database in the collection, which is **gone** because we **restart the container**. 

>[!tip]
>When you restart a container, everything that you configured in that container application is gone. Data is lost. There is **no data persistence in the container** itself.

**Volumes** makes it possible to have persistency between the container restarts.

So let’s create a DB called `my-db` and inside it create a collection called `users`. Then start the application with `node server.js` and navigate to [localhost:3000](http://localhost:3000) to get the User Profile page of the JS application. Here let’s make some modification and click on “Update Profile”:

![Untitled](Other/attachments/Untitled%2054.png)

Now, go back in MongoExpress UI by  [localhost:8080](http://localhost:8080) and you can see the updated entry:

![Untitled](Other/attachments/Untitled%2055.png)

This means the connectivity with MongoDB works.

Now, stop the containers with `down` command of docker compose tool:

```powershell
docker-compose -f mongo.yaml down
```

This command removes the Docker Network as well as the containers.

---

# Dockerfile - Building our own Docker Image

We have this scenario: you developed an application feature you have tested it and now you are ready to deploy it. To deploy your application, it should be packaged into its own Docker Container (first line of the picture above). This means we’re gonna build a Docker Image from our JS-NodeJS backend application and prepare it to be deployed on some environment (second line):

![Untitled](Other/attachments/Untitled%2056.png)

Taking the big picture of the entire project, we have developed our JS application and used MongoDB docker container to use it and now it’s time to commit it to Git.

![Untitled](Other/attachments/Untitled%2057.png)

In this case, we’re going to simulate these steps on the local environment. But still, we’re going to show how these steps actually work. So, after commit you have a continuous integration that runs. What does actually Jenkins do with this application? When it builds the application, it packages it in a Docker Image and  then pushes it into a repository.

![Untitled](Other/attachments/Untitled%2058.png)

So, we’re going to simulate what Jenkins does with their application and how it actually packages into a Docker Image on the local environment. So we’re going to do all this on our laptop, but it’s basically the same thing that Jenkins will do.

**What is a Dockerfile?**

In order to build a Docker Image from an application, you have to copy the contents of that application into the Dockerfile. In order to do that, we’re going to use a blueprint for building images, which is called Dockerfile.

![Untitled](Other/attachments/Untitled%2059.png)

Let’s see what is a Dockerfile and how it looks like.

>[!top]
💡 Dockerfile is a blueprint for creating Docker Images.

The syntax of Dockerfile is super simple.

The first line of every Dockerfile is `FROM [*image*]`. So whatever image you are building, you always want to base it on another image. In our case we have a JS application with NodeJS backend, so we are going to need a `node` inside of our container so that it can run our node application instead of basing it on a Linux Alpine or some other low-level image, because then we would have to install `node` ourselves on it.

![Untitled](Other/attachments/Untitled%2060.png)

So, go to DockerHub site and find `node` and you can see that there is a ready `node` image that we can base  our own image from:

![Untitled](Other/attachments/Untitled%2061.png)

We take the latest version. So the first line of the Dockerfile

```dockerfile
FROM node
```

means that we’re going to have `node` installed inside of our image. So when we stat a container and we actually get a terminal on the container, we can see that node command is available.

Next, we configure the **environmental variables** inside the Dockerfile. We have already done this using the docker-compose, so this will be just an alternative to defining environmental variables in a docker-composed. Maybe it’s better to define the environmental variables externally in a docker-compose file because if something changes, you can actually override it instead of rebuilding the image. But this is an option.

![Untitled](Other/attachments/Untitled%2062.png)

Next, there is `RUN` command. With this you can execute ant kind of Linux commands. In this case you see `mkdir` that creates a /home/app directory. 

![Untitled](Other/attachments/Untitled%2063.png)

Note that this directory is going to live **inside the container**, not on my laptop.

Next, we have `COPY` command. Can we execute a Linux copy command using RUN? Yes, but the difference here is that all these commands (RUN for example) get executed **inside of the container**; on the other hand the COPY command actually **executes on the host**. In out case, the first parameter of the COPY command ( `.` ) is the source and the second one ( `/home/app`) is the target.

![Untitled](Other/attachments/Untitled%2064.png)

Finally, the `CMD` command executes an entrypoint Linux command. This is the same as we did in the terminal window on the laptop, but now it executes inside the container:

![Untitled](Other/attachments/Untitled%2065.png)

Remember that we are able to do that because we are basing on the node image that already has node preinstalled.

Again, what is the difference between `RUN` and `CMD`? CMD is an **entrypoint command**, so you can have multiple RUN commands with Linux commands but just one of CMD command.

![Untitled](Other/attachments/Untitled%2066.png)

Let’s create the Dockerfile. Like docker-compose file, the Dockerfile is part of the application code, so let’s create a new file inside the app folder:

```dockerfile
FROM node:13-alpine

ENV MONGO_DB_USERNAME=admin \    
	MONGO_DB_PWD=password

RUN mkdir -p /home/app

COPY ./app /home/app

CMD ["node", "server.js"]
```

Since we saw that Dockerfile is a blueprint for any Docker Image, that means that every Docker image that there is on DockerHub should be on its own Dockerfile. Indeed if we find a specific version of node on the DockerHub, for example the following one:

![Untitled](Other/attachments/Untitled%2067.png)

and click on it:

![Untitled](Other/attachments/Untitled%2068.png)

you can see that every image is based on another base image.

So in order to actually visually comprehend how these layers stacking works with images let’s consider this simplified visualisation:

![Untitled](Other/attachments/Untitled%2069.png)

The Dockerfile has to be named exactly with `Dockerfile` name.

![Untitled](Other/attachments/Untitled%2070.png)

Now that we have a Dockerfile ready, let’s see how to actually use it, so how do we build an image out of it.

Note that for everything to work well, the application project structure must be as follows:

![Untitled](Other/attachments/Untitled%2071.png)

In order to build an image using a Dockerfile, we have to provide two parameters:

- image name with `-t` with a tag (a version)
- location of the Dockerfile

In our case we have:

```powershell
docker build -t my-app:1.0 .
```

Note that we use `.` because we are in the same folder of the Dockerfile with the terminal. After running that command, we’ll get:

![Untitled](Other/attachments/Untitled%2072.png)

(*this is a screenshot from the video tutorial. In that case node:13-alpine was already on the laptop. In our case, we would have seen logs for pulling node:13-alpine from DockerHub*)

![Untitled](Other/attachments/Untitled%2073.png)

Going back the the high-level picture of the project, we’re just simulated what Jenkins server does. What Jenkins server does is it takes the Dockerfile that we created, so we have to commit the Dokerfile into the repository with the code, and Jenkins will then build a Docker Image based on the Dockerfile.

![Untitled](Other/attachments/Untitled%2074.png)

Note that you don’t develop alone, but you are in a team, so other people may want to have access to the up to date image of the application. In order to do that, you have to actually share the image so it is pushed into a Docker Repository and from there other people can take it; for example, a tester wants to download the image from there and test it locally, or a development server can actually pull it from there.

Now, let’s run the container from the new image with:

```powershell
docker run my-app:1.0
```

But this returns an error:

![Untitled](Other/attachments/Untitled%2075.png)

Why this error? Because we’re not telling it to look in the correct directory. Since in the Dockerfile we are telling to copy all the resources in the /home/app directory, I have to adjust the `CMD` command:

```dockerfile
FROM node:13-alpine

ENV MONGO_DB_USERNAME=admin \    
	MONGO_DB_PWD=password

RUN mkdir -p /home/app

COPY . /home/app

CMD ["node", "/home/app/server.js"]
```

**IMPORTANT!** Whenever you adjust a Dockerfile, you have to **rebuild the image** because the old image cannot be overwritten. So, before removing the wrong image, you have firstly to remove the container using that image. So, find the container ID using `docker ps -a | grep my-app` and then:

```powershell
docker rm 3c58e681cc8c5
```

then find the image ID with `docker images` and then remove it with `docker rmi` command:

```powershell
docker rmi 2e0a 2e0a4d16e074
```

Now, you can rebuild the image with `docker build -t my-app:1.0` and let’s create a container from that image with `docker run my-app:1.0`. Now it works and you can find the new container with `docker ps`.

Now let’s enter inside the container with:

```powershell
docker exec -it 51c6912d69f5 /bin/sh
```

or `docker exec -it 51c6912d69f5 /bin/bash` (find the one that works)

The cursor changed and this means we are inside the container; in particular we are in the root directory and with `ls` you can see the virtual filesystem:

![Untitled](Other/attachments/Untitled%2076.png)

Let’s check some of this stuff.

Firstly, let’s check environmental variables. Hit `env`:

![Untitled](Other/attachments/Untitled%2077.png)

`MONGO_DB_USERNAME` and `MONGO_DB_PWD` are the ones that we set; the other ones are set by default.

Next, let’s check the directory `/home/app` because we created this directory and copied all the project content inside that directory:

![Untitled](Other/attachments/Untitled%2078.png)

![Untitled](Other/attachments/Untitled%2079.png)

Actually, in this container directory we only need the JS files or the artifacts (if we build the JS application artifacts), not the Dockerfile or docker compose files. So let’s improve that.

Let’s create a subfolder called app in my-app folder and let’s copy inside it just files that we need for starting the application inside the container that are the ones selected in the following:

![Untitled](Other/attachments/Untitled%2080.png)

Now, instead of copying the entire directory into the `/home/app` folder, we’ll copy only the `app` folder:

```dockerfile
FROM node:13-alpine

ENV MONGO_DB_USERNAME=admin \    
	MONGO_DB_PWD=password

RUN mkdir -p /home/app

COPY ./app /home/app

CMD ["node", "/home/app/server.js"]
```

Again, since we modify the Dockerfile, we need to recreate the docker image. So, as we did before, stop the running container, remove it, then remove the image, then build again the new image, then create the new container. Now let’s enter the container again, and check with ls `/home/app`:

![Untitled](Other/attachments/Untitled%2081.png)

Now we just have the needed files.

Note that we copied this files because we have just few files, but usually if you have a huge application you would want to compress them and package them into an artifact and then copt that artifact into a docker container.

# Private Docker Repository - Pushing our built Docker Image into a private Registry on AWS

video

# Deploy our containerized app

video

# Docker Volumes - Persist data in Docker

**Data Volumes** are used for data persistence in Docker.

When do we need Docker Volumes?

A container runs on a host, we have a DB container. A container has a virtual file system where the data are usually stored. But here there is no persistence, so **if I were to remove the container or stop it and restart it, the data in this virtual file system is gone**.

On a host we have a physical file system and the way **Volumes** work is that we plug the physical file system path into the container file system path. So with simple words, a directory folder on a host file system is mounted into a directory folder in the virtual file system of the container:

![Untitled](Other/attachments/Untitled%2082.png)

What happens is that when a container writes to its file system, it gets **replicated or automatically written on the host file system directory and vice versa**.

There are different types of Docker Volumes, so different ways of creating them.

### Host Volumes

Usually the way we create Docker volumes is using `docker run` command with the option `-v`, with which we define the connection between the host directory and the container directory:

```docker
docker run -v /home/mount/data:/var/lib/mysql/data
```

![Untitled](Other/attachments/Untitled%2083.png)

The main characteristic of the Host Volume is that you decide where on the host file system that references made, so which folder on the host file system you mount into the container.

### Anonymous Volumes

You can create a Volume just by referencing the container directory, so you don’t specify which directory on the host should be mounted, but that’s taking care of the docker itself. So that directory is, first of all, automatically created by Docker under the /var/lib/docker/volumes/, so for each container there will be a folder generated that gets mounted automatically to the container:

![Untitled](Other/attachments/Untitled%2084.png)

They’re called anonymous volumes because you don’t have a reference to this automatically generated folder, basically you just have to know the path.

### Named Volumes

They’re actually an improvement of the anonymous volumes and it specifies the name of the folder on the host file system and the name is up to you. It just to references the directory. So in this case compared to anonymous volumes, you can actually reference that volume just by name, so you don’t have to know exactly the path.

![Untitled](Other/attachments/Untitled%2085.png)

Among these three types, the mostly used ones and the one that you should be using in production is actually the **Named Volumes**, because there are additional benefits to letting Docker actually manage those volumes directories on the host.

We saw how to create volumes with `docker run` command. If you want to use `docker-compose`, the method is actually the same.

![Untitled](Other/attachments/Untitled%2086.png)

We must specify `volumes` voice both at the container level and at the `services` level:

- Under `mongodb` voice, we have to define the **named volumes** as we did with docker run command, so by specifying under which path that specific volume can be mounted.
- At the very end of the docker compose file, at the same indentation level of `services` you have to list all the volumes that you have defined. So if you were to create volumes for different containers, you would list them all here.

The benefit of Named Volumes is that you can actually mount a reference on the same folder on the host to more than one containers, and that would be beneficial if those containers need to share the data. In this case, you would want the same volume to two different containers and you can mount them into different path inside the container.

# Volumes Demo - Configuring persistence for our Demo Project

Let’s start the MongoDB with docker compose. Let’s call this file `docker-compose.yaml`:

```yaml
version: '3'
services:
  mongodb:
    image: mongo
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
  mongo-express:
    image: mongo-express
    ports:
      - 8080:8081
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_MONGODB_SERVER=mongodb
```

Let’s run both `mongodb` and `mongo-express` containers:

```powershell
docker-compose -f docker-compose.yaml up
```

If we navigate to [localhost:8080](http://localhost:8080) we’ll see the MongoExpress UI we have just seen in the past. Let’s create again a `my-db` database and a `user` collection inside of it.

(*of course the connection between `my-db` and the application need to be set inside the `server.js` file, but I don’t care about it at the moment because it’s not the purpose of mine right now. But if you want to see that take a look at the code on Gitlab*)

Let’s run the application with `npm run start`, then go to [localhost:3000](http://localhost:3000). Make an edit on the User Profile page. Now if I were to restart the `mongodb` container, I’ll lose all the data inside the DB.

So because of that we’re going to use **Named Volumes** inside of the `docker-compose.yaml` file to persist the data in the MongoDB.

```yaml
version: '3'
services:
  mongodb:
    image: mongo
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
		volumes:
			- mongo-data:/data/db
  mongo-express:
    image: mongo-express
    ports:
      - 8080:8081
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_MONGODB_SERVER=mongodb
volumes:
	mongo-data: 
		driver: local
```

In the last `volumes` voice we put all the volumes that I need in any containers, and since we need data persistency for MongoDB, we created `mongo-data`, that is the name of the volume reference, but we also need to provide a `driver`, in particular we set `local`, meaning that the actual storage path, that we’re going to see later once it’s created, is actually created by Docker itself. And `driver: local` is an additional information for Docker to create that physical storage on a local file system.

Once we have a nave reference to a volume, we can actually use it in the container, and we put that at the `volumes` voice under the `mongodb` voice. In particular, `mongo-data:/data/db`, where `mongo-data` is the host-volume-name and `/data/db` is is the path-inside-of-the-container. Indeed if you check online, the default path where MongoDB stores its data is `/data/db` and we can check it out:

![Untitled](Other/attachments/Untitled%2087.png)

For MySQL is `var/lib/mysql`, while for Postgres is `var/lib/postgresql/data`.

Now let’s restart docker-compose and create a database and a collection and make a change in the User Profile of the application, as we did before. But this time, if I were to restart all these containers, these data should be persisted.

Now let’s see where the Docker volumes are located on the local machine. This differs among different OS:

![Untitled](Other/attachments/Untitled%2088.png)