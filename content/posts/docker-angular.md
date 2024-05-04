+++
title = 'Docker Alpine + ExpressJS + Angular'
date = 2024-05-04T18:13:22+03:00
cover = 'images/docker-angular-cover.jpg'
draft = false
+++

This was for a project that I did recently that required a reporting dashboard app to be served from a docker image, and would live inside small low-powered linux devices. 
The app had to fulfill these criterias:
It needs to be replicated quickly for many devices
It needs to be small, there isn't a lot of space
It needs to be able to connect to MariaDB database in the localhost network
Certain fields are configurable from a JSON database that is exposed outside the container.Docker container containing an ExpressJS app that serves an Angular app


Docker
1. It needs to be replicated quickly for many devices
Docker has been around for a considerable time already, released on 2013, it aims to solve the problem of "But it works on my computer". 
A popular use of docker is to pair it with kubernetes to spin up multiple microservices to handle large spikes of concurrent users, and then remove them after the spike to save resources. Each microservice is a tiny server on its own. 
Since docker images are essentially like a backup copy of a tiny server, it will always run the same way without much configuration. Which makes it easy to copy the image to any new device, and spin up a new docker container running the app. 
Architecture Solution
Use Docker to containerize the app
ExpressJS serves as a web server for the frontend Angular app.
ExpressJS uses lowdb to read/write configurations to a JSON file in a folder called data/.
ExpressJS uses knex.js to query SQL database.
The final work folder looks something like this – ```
backend/
    |- src/...
    |- dist/
        |- server.min.js
frontend/
    |- src/...
    |- dist/
        |- public/
            |- main.min.js
            |- vendor.min.js
            |- index.html
docker/
    |- data/
        |- local-db.json
    |- server.min.js
    |- package.json
    |- public/
        |- main.min.js
        |- vendor.min.js
        |- index.html
    |- Dockerfile
    |- .dockerignore
```



Both frontend and backend is minimized to reduce space. The contents of the the minimized code is copied to the docker folder so that it is easier for me to pack it into a docker image. 
Express server is serving the public folder (Angular app) under root route. While the rest of the API starts with /api/ prefix. This also means that Angular can use /api when calling http, so it'll always call something in the same domain. this.http.post('/api'+ reportUrl);


Package.json from the backend is included as well, because I need install the node_modules required for the ExpressJS app. It's probably possible to somehow bundle the dependencies using Webpack, but I was strapped for time and didn't investigate further how to do so. ```prettyprint lang-typescript
import * as FileAsync from 'lowdb/adapters/FileAsync';
const low = require('lowdb');

low(new FileAsync(path)).then(db => {
    db.defaults({
        maxEntries: MAXENTRIES,
        configs: []
    })
    .write();
});
```


This will write a default JSON configuration to data/local-db.json.
Dockerfile
2. It needs to be small, there isn't a lot of space
A common practice during development is the throw all the source files into the docker container and run a full node.js server to serve your application. It's the easy, fast, painless. 
But a full node.js on debian server is quite big - nearly 650MB from the start. After installing our ExpressJS app and Angular it went to about 1.3GB. 
So we opted to go with the new barebone OS based on Alpine linux, which is a huge 50MB. With everything installed, it came up to about 300MB. Still much smaller than the full server image. 
The full list of available node base images are here.
A barebone OS means that we need some extra files in order to install the node modules. ```prettyprint lang-bash
FROM node:10.15.3-alpine

ENV JWT_SECRET 512FF7B2EMCKAHG24C
ENV BCRYPT_SALT 12
ENV PORT 8080
ENV TOKEN_EXPIRES_IN 24h

WORKDIR /usr/src/app
COPY . .
RUN apk --no-cache add --virtual native-deps \
    bash g++ gcc libgcc libstdc++ linux-headers make python && \
    npm install --quiet node-gyp forever -g &&\
    npm install --production --quiet
EXPOSE 8080
CMD forever server.min.js 8080
```


There is a very detailed documentation on how to write a Dockerfile, but essentially a docker file is similar to a bash script that runs commands in sequence and finally serve the app. `FROM node:10.15.3-alpine` is how I select the image as a base for the container. 

I'm setting environment variables with the `ENV` command. Which can then be read in the ExpressJS app via `process.env`. 

`WORKDIR` selects the current folder I want to use IN the container. If no directory exist, Docker creates them. 

I include several installs (gcc, python etc) that is required for node to be built. 

And then finally serving `server.min.js` with [Forever](https://github.com/foreverjs/forever).


Forever is a CLI tool that will run the node server as a daemon, and cannot be killed accidentally. 
Docker Run
3. It needs to be able to connect to MariaDB database in the localhost network
Now comes the fun part – running the app. First we must build it. ```
$ docker build --tag node-app:alpine .
```

Docker build reads the command from Dockerfile and builds an image. 

`--tag` allows me to name the docker image, so that it's easier to find it later. 

After building, `docker images` shows that I've indeed built an image

```
$ docker images

REPOSITORY     TAG          IMAGE ID         CREATED          SIZE
node-app       alpine       f4ca2e0425a0     3 days ago       354MB
```


Now to run the docker image```
$ docker run \
    --name node-app \
    --network host \
    --restart 'unless-stopped' \ 
    --detach \
    node-app:alpine
```


`--name` lets me set a custom name. If it's not provided, docker will create a name for you like precious-grauss or similar. 

`--restart` will make the running container run forever unless specifically stopped. 

`--detach` will make it run in the background, freeing up my terminal for other things. If you don't use this, you'll need a 2nd terminal to stop your docker container.

`--network` is the *key option* here. One of the requirements is that the docker container should be able to connect to a database on localhost. And by using the option `host`, we can do so. There are other methods of creating docker networks and connecting containers together. You can read more [here](https://docs.docker.com/network/). 

With this command, I am now able to see the app run on `localhost:8080`. And my app is able to connect to the database successfully. 🎉

<iframe src="https://giphy.com/embed/4apcfHxnRfwjK" width="480" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/dance-learn-wiig-4apcfHxnRfwjK"><small>via GIPHY</small></a></p>


Docker volume
4. Certain fields are configurable from a JSON database that is exposed outside the container.
The docker container should now be running smoothly, until, I find that the local-db json isn't really updated. And I'm not able to control the configuration from outside the container. That's because I left out the `--volume` option during docker run that allows me to mount a folder unto docker, which can then be shared between host machine and the docker container. You can read more about data persistence in docker [here](https://docs.docker.com/storage/).

```
$ docker run \
    --name node-app \
    --volume $(pwd)/data:/usr/src/app/data \
    --network host \
    --restart 'unless-stopped' \ 
    --detach \
    node-app:alpine
```

`$(pwd)` will insert the current working directory into the command. If I were to call docker from a different folder, the pwd will be different. 

Here the volume command mounts the data folder to the data folder that exists inside the docker container `/usr/src/app/` folder. If the target folder does not exist, `--volume` will create it for me. 

Run this, and now I can control the configs from outside. 


Saving and Loading images
Now that I have the image up and running. I need to export the image into a format I can import into different devices and start up the same instance. ```
$ docker save -o ./node-app-image.tar node-app:alpine
```

The `save` command exports the current image state into a tarball. I make sure that I have not added any custom configuration, because I don't want them saved into the image. 

```
$ docker load -i ./node-app-image.tar
```

I then run the `load` command after I have downloaded the tarball into the machine. Loading the image will include the image into `docker images`. And then I do the `run` command as before, and it should find the same `node-app:alpine` image I saved. 



Thoughts
Docker is really quite powerful. It has let developers scale up apps instantly and confidently, because the environment will always be exactly the same. 
That said, I think the docker documentation can be a little easier to deal with. There is just such a huge amount of configuration to learn, it is no wonder docker setup is a career on its own. 
Regardless, learning docker is exciting. Let me know if there is a better way!<iframe src="https://giphy.com/embed/xUA7bfzccySdtHWUUg" width="480" height="360" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/cute-summer-beach-xUA7bfzccySdtHWUUg"><small>via GIPHY</small></a></p>

Resources
---------
[1] [https://github.com/nodejs/docker-node](https://github.com/nodejs/docker-node)
[2] [https://github.com/nodejs/docker-node/issues/282](https://github.com/nodejs/docker-node/issues/282)
[3] [https://stackoverflow.com/questions/43316376/what-does-net-host-option-in-docker-command-really-do](https://stackoverflow.com/questions/43316376/what-does-net-host-option-in-docker-command-really-do)
[4] [https://stackoverflow.com/questions/12701259/how-to-make-a-node-js-application-run-permanently](https://stackoverflow.com/questions/12701259/how-to-make-a-node-js-application-run-permanently)



