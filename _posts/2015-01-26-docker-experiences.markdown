---
layout: post
title: "Docker experiences"
date: 2015-01-26T12:10:31+00:00
comments: true
---

Some thoughts on my experiences with docker.

# Intro #

Lately my friend convinced me to enter the Docker bandwagon, and for the last few days I've read about most of the tools on Docker out there. Turns out I should have started from the basics, although [tutum.co](https://tutum.co) seems very nice, once it is production-ready. [fig](http://fig.sh) and [dokku](http://dokku.com) also look cool.

But I started from scratch.

## Docker ##

Docker terminology is a PITA. Here's my mental guide

* **images** are like vm images, which you customize mostly through a `Dockerfile`
	* these are built in a tree-like, so docker reuses "checkpoints" for images that share something (eg: nginx and mongo based on ubuntu: ubuntu will be shared, so you just download ubuntu one time)
* **images** are normally tagged for easy identification
	* an example is the `dockerfile/java` image that has tags for (oracle|open)jdk6/7/8
* **containers** are processes that run images
* you can run **containers** in daemon mode, but for getting started I found it's easier to run and `Ctrl+C` to quit it, otherwise you'll have a bunch of running **containers** without noticing
* you should read the Docker docs carefully.

### Why Docker ###
You'll get a better answer elswhere, but for me it's:

* isolation (and security, and multiple versions etc etc)
* easy replicability(is this an english word?) of images across machines
* scalability is nice, but not for small apps like mine
	* frameworks like [deis](http://deis.io), [flynn.io](https://flynn.io/), etc promess to easily scale a cluster up and down through the use of technologies such as docker

### Docker concepts and tricks ###

####You should (generally speaking):

* either build an image from a `Dockerfile`, or pull an image from docker index
* use a container per service. Example:
	* 1 container for mongo
	* 1 for nginx
	* 1 for java running tomcat or whatever
* the rationale for this is, generally speaking, separation of concerns.
* so, you should use a different image for each service; although I'm also running eg: wordpress from a single docker image (includes `wordpress + mysql`). I'm a practical guy.
* share data with docker volumes: `docker --v src_folder:target_folder_in_container`
* either link containers to expose eg: mongo to the main app container through `docker run --name src` and `docker run --link src:src` or use some kind of Docker ambassador pattern
	* A stupid thing is that if you remove an image through its id, the name stays behind. So if you ran an image named `web`and stopped the container, you  will have to do `docker rm web` before starting another container with the same name
* `docker run -d` dameonizes the container, I only pass the `-d` flag when I'm sure that the command is correct
* If you're running docker through eg: git hooks, don't use `docker -t`, otherwise the script won't wait for the docker command to finish

####You can (you should know this):

* enter a running container through `docker exec -t -i CONTAINER_ID /bin/bash`
* enter a stopped container through `docker run ... --entrypoint=/bin/bash ...`, regardless of the containers `CMD` or `ENTRYPOINT`
* parse `docker inspect`with [jq](http://stedolan.github.io/jq/manual/)

### Deploying an app ###

For this example I'll be using CoreOS on Digital Ocean.
Here's my referrer if you want to give it a try: [https://www.digitalocean.com/](https://www.digitalocean.com/?refcode=1bfa1739fe3c)

My app consists of

* a mongodb image
* an nginx image
* a clojure image, this is essentially an image to run a fatjar

Additionally, I needed

* a socat image
* a lein image
 * careful, see [http://www.martinklepsch.org/posts/running-a-clojure-uberjar-inside-docker.html](http://www.martinklepsch.org/posts/running-a-clojure-uberjar-inside-docker.html)
* the lein image would later become a leiningen/node/bower/maven image, because my app is clojure on the backend, angular on the frontend with a node/bower toolchain to build the js.

Let's get these points one by one.

### MongoDB ###

We want to run mongodb, but we want the data out of the container. For that, we'll use docker volumes.
We also want to expose the mongod service to other containers. For that, we'll use container linking.
Your preferred image should expose a volume, check the image's Dockerfile:

	VOLUME ["/data/db"]

By now you should either have pulled a mongo image or created your own Dockerfile and an image based on it.

	docker run -P -v /home/mongo/data:/data/db --name mongo some/mongo

If your dockerfile has `EXPOSE` instructions, the `-P` will randomly expose the ports on the **host** container.
Don't worry you only need to know the port if you want to connect to it.
If you're running the latest docker, you can enter this container with `docker exec` and inspect the db. Otherwise, you can try some tutorials to set up `nsenter` or simply connect through the exposed ports:

    3ce6046e2ee4        mping/mongo:latest        "mongod"               44 hours ago        Up 44 hours         0.0.0.0:49155->28017/tcp, 0.0.0.0:49156->27017/tcp   mongo

In this example you have local ports `49155, 49156` binded to the container's `28017,27017`ports which are mongo ports. I'm guessing the docker way would be to launch another mongo container and connect to the server, but I just use `docker exec -it 3ce6046e2ee4 /bin/bash` and voilá.

### App ###

Assuming we have a docker image for our app (more on this later), we just need to run it and instruct docker to use the mongo service. Notice that the `DATABASE_URL` is set to an address called `mongo`. This will work because when you link docker containers, the `/etc/hosts` file has an entry that sets the proper ip for the `name`.

	 docker run -p 3000:3000 \
		--name web \
		--link mongo:mongo \
		-h mysite.com \
		-e DATABASE_URL=mongodb://mongo:27017/myapp \
		-e ENV=prod \
		mydocker/image

### nginx ###

Assuming you have a proper nginx image, you just need to run it and link it to the conf files.
The interesting bit is setting an upstream to `web` in `sites-available`; remember that when you link containers, docker will update the container's `/etc/hosts` with the ip for the linked container. So `web` will resolve to the running container named `web`.

	upstream app_server {
			server web:3000;
	}

Afterwards, just run it:

	docker run -d -h mysite.com -p 8888:80 -p 4443:443 \
		--link web:web \
		-v `pwd`/nginx/sites-enabled:/etc/nginx/sites-enabled \
		-v `pwd`/nginx/certs:/etc/nginx/certs \
		-v `pwd`/nginx/logs:/var/log/nginx \
		local/nginx


What's all this hassle for? Well, for the most of it, you get to have docker images that are ready to run and are not specifically configured to a particular configuration. You can spin another mongo or nginx and use it in different projects, as long as the volumes and links are correct.

## CoreOS ##

CoreOS on DO has a fedora image that you can run, which has most of linux utilities (eg: nano).
It's available on `/usr/bin/toolbox` and it mounts the whole drive as a docker volume under `/media/root`.

Docs are here: https://coreos.com/docs/cluster-management/debugging/install-debugging-tools/


## Clojure ##

I'm experimenting with `lein uberimage`, a plugin that generates a docker image for your uberjar.
The main advantage of having an image, instead of building a jar and generically running it, is that you get your own app image that can be pushed to a repo, rolled back, etc.

Unfortunately, I hit alot of roadblocks:

 * The `uberimage` plugin expects a docker API on localhost:2375; on my dev machine (ubuntu), I had to update the docker daemon to run on `127.0.0.1` aside with the unix socket. Careful not to expose the docker api on all interfaces (don't use `0.0.0.0`)

		less /etc/init/docker
		...
		DOCKER_OPTS="--host=tcp://localhost:2375 --host=unix:///var/run/docker.sock"
		...

 * If you're building an image from a container (CoreOS forces you to do everything from a container), you'll need to communicate with the host docker api to let the leiningen plugin push the image. I had to install a `socat` image and expose it with a name.
This is actually a good practice that allows you to expose the host docker api to a give container, either by exposing the socket through volumes or the http api through the named container `docker-http`

 * docker [socat](https://github.com/sequenceiq/docker-socat/blob/master/start)

 * run the image in the background

		docker run -d -p 2375:2375 --volume=/var/run/docker.sock:/var/run/docker.sock --name=docker-http someimageof/lein
		...

 * when specifying the docker api, use the named docker:
		docker run ... someimageof/lein uberimage -H http://docker-http:2375 -t mydocker/image

		docker run -i -t --link docker-http:docker-http -v /var/run/docker.sock:/tmp/docker.sock:rw -v /home/core/ezcode.deploy:/project someimageof/lein uberimage -H http://docker-http:2375 -b someimageof/jdk8 -t mydocker/image

### But... But... ###
I lost the whole weekend trying to run a `!"#&"@` jar. The fix that worked for me was this:

	java -Xmx512m -noverify -jar -Djava.security.egd=file:/dev/urandom target/myjar-standalone.jar

So how did I got there? First I used `jps` to see where the launching was being stuck:

	jps -l
	15990 sun.tools.jps.Jps
	15980 target/ezcode-0.1.0-standalone.jar


Then I used `jstack` to inspect the stack of the running java application:

	jstack 15980
		...
		java.lang.Thread.State: RUNNABLE
			at java.io.FileInputStream.readBytes(Native Method)
			at java.io.FileInputStream.read(FileInputStream.java:220)
			at sun.security.provider.SeedGenerator$URLSeedGenerator.getSeedBytes(SeedGenerator.java:493)
			at sun.security.provider.SeedGenerator.generateSeed(SeedGenerator.java:117)
			at sun.security.provider.SecureRandom.engineGenerateSeed(SecureRandom.java:114)
			at sun.security.provider.SecureRandom.engineNextBytes(SecureRandom.java:171)
			- locked <0x00000007d8b0ee08> (a sun.security.provider.SecureRandom)
			at java.security.SecureRandom.nextBytes(SecureRandom.java:433)
			- locked <0x00000007d8b0ee60> (a java.security.SecureRandom)
			at java.security.SecureRandom.next(SecureRandom.java:455)
			at java.util.Random.nextInt(Random.java:189)

Some thread on some `secureRandom` call or whatever caught my eye. Googling about it, I found this: http://docs.codehaus.org/display/JETTY/Connectors+slow+to+startup which had the workaround at the bottom:

	Set the system property: -Djava.security.egd=file:/dev/urandom
	Make sure that the java.security file contains this setting:
		securerandom.source=file:/dev/urandom

And this only worked on jdk8. I couldn't even apply this in jdk7.
