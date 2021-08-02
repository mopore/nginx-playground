NGINX Playground - Reverse Proxy
================================

## What is it about
This project demonstrates how to setup a nginx as a docker container to redirect path-specific 
http requests to a "background container" not running on a port 80.
While using docker-compose and the default network, the above setup will create the only way the "background container" will be accessible outside of the docker network.

In concrete, all requests to http://localhost/ch/api/ will be redirected to container named "pan-hr-api-ch" running on the internal port 8081. A request to http://localhost will lead to a default nginx welcome page (index.html).

## Prerequisites
You will need a docker and docker-compose installed on your system.
To run this project as it is, you will need to have a docker image 'pan-hr-api-ch' which is not 
public. For this demonstration, you can adapt the project (given the description) and create your own "background container".

## How to us it
Simply run `start.sh` and `stop.sh` to start and stop the containers. Note that the referred image "pan-hr-api-ch" needs to be available. Otherwise, you have to provide your own.

## How to provide your own "background container"
The following description uses an example with node.js. It is expected that the reader knows how to use docker and how to create docker images.

### nginx Configuration
Taking a look into the http section of the [nginx configuration (config/nginx.conf)](/config/nginx.conf):

```
http {
	server {

		listen 80;
		server_name localhost 127.0.0.1;

		location / {
			root /usr/share/nginx/html;
			try_files $uri /index.html;
		}
		
		# Pass all incoming request to '/ch/api' to our docker local ch HR API
		location /ch/api {
			proxy_pass			http://pan-hr-api-ch:8081;
			proxy_set_header	X-Forwarded-For $remote_addr;
		}
	}
}
```
We have to ensure that the background container will be reachable under the name `pan-hr-api-ch` and 
the port `8081`. Further, the container has to provide a path `/ch/api`.
The name of the hostname will be defined in the docker-compose file. Providing the correct port and path is the task of the container itself.


### Docker Compose Configuration
Taking a look into the [docker compose configuration](docker-compose.yml):
```
version: "3"
services:
  pan-proxy:
    container_name: pan-proxy
    image: nginx:latest
    depends_on:
      - pan-hr-api-ch # References the service name.
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf
      - ./content:/usr/share/nginx/html
    ports:
      - 80:80 # Publishs only port 80 outside of docker compose context.

  pan-hr-api-ch:
    container_name: pan-hr-api-ch # Sets the docker internal hostname to be called on http://pan-hr-api-ch.
    image: pan-hr-api
    environment:
      - CONFIG_FILE_ENV=ch-jni-local # The config file instructs to provide an URL following "/ch/api/...".
      - API_PORT=8081 # Will start HR API on port 8081 in docker compose context.
      - TZ=UTC # Sets the timezone to UTC
```
Here you will find the service config for the nginx container. We see that the nginx container depends on our "background container" named "pan-hr-api-ch". Also, you can see the volume definitions to read our custom nginx configuration and our custom index.html by referring to our local content folder.

In the section below, you have the service definition for "pan-hr-api-ch" (our background container).
"container-name" directly defines the hostname inside the docker network.
Our background container also comes with environment variables that will be read inside the container to set up the right configuration ("CONFIG_FILE_ENV"), define the internal port ("API_PORT") and set the timezone ("TZ").


### Provide the right port to run your container
As you remember, the nginx configuration avoids the container on port 8081. This port number is redundantly defined as an environment variable ("API_PORT") for the container that can be read out to launch the server, as shown in the code below: 

![Providing the correct port](/images/code_port.png)

*Note:* For testing purposes, use an extra line `port` in your docker-compose config of your "background container" to test if your container properly starts and is reachable.

### Provide the URL path you want the redirect to go to
Inside the container application, provide the path ("/ch/api"), which is expected from the nginx configuration. 
In the example below the path is compiled by a country code ("ch") and "api" plus further additions like "test" to provide a quick "is alive" test call. The country code is delivered within a country-specific configuration file. To point the application to the correct config file, the environment variable "CONFIG_FILE_ENV" has been passed.

![Providing the correct path](/images/code_path.png)