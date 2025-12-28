# Traffic Light Docker Challenge

## Part 1: Containerizing the Applications

Each color has its own folder with an `app.js`, `package.json`, `index.pug`, and a `Dockerfile`. The apps are basic Express servers that display the color.

### Step 1: Modify the Dockerfile

Open the `Dockerfile` in each color's folder (red, yellow, green).

1) Modify it to use `node:20-alpine` as the base image.

2) Create a working directory `/app` inside the container.

3) Copy the `package.json` file to the working directory.


#### Checkpoint 1

Try building the image for one color, say red. Tag it as `lights:red-v0.0.1`.

```
docker build -t lights:red-v0.0.1 ./red
```

Verify the image was created:

```
docker images
```

You should see the new image listed.

### Step 2: Install Dependencies and Copy Files

1) Add to the Dockerfile the `RUN` command to install dependencies using `npm install`.

2) Copy the `app.js` file to `/app`, and `index.pug` and `favicon.ico` to `/app/views`.


#### Checkpoint 2

Build the image again, tagging it as `lights:red-v0.0.2`.

Run a container from this image with an interactive shell to verify the folder structure:

```
docker run -it --rm lights:red-v0.0.2 sh
```

Inside the container, check the structure:

```
ls -la
ls -la views/
```

You should see:

- /app
  - app.js
  - package.json
  - package-lock.json
  - node_modules/ (with dependencies)
  - views/
    - index.pug
    - favicon.ico

Exit the container.

### Step 3: Expose Port and Set CMD

Add `EXPOSE 80` to the Dockerfile.

Set the CMD to run the app: `node app.js` (Remember CMD's syntax!)


### Checkpoint 3

Build the image again, tagging as `lights:red-v0.0.3`.

Run the container, mapping port 80 to 80 on host:

```
docker run -d -p 80:80 --name red-app lights:red-v0.0.3
```

Open your browser to `http://localhost:80` and verify the red light app works.

<br>

**If it works,** retag the image to `lights:red-v1.0.0`:

```
docker tag lights:red-v0.0.3 lights:red-v1.0.0
```

Stop and remove the container:

```
docker stop red-app
docker rm red-app
```

<br>

Copy the final Dockerfile to the yellow and green folders, and build their images similarly.

<br>
<br>


## Part 2: Networking Containers

Now that we have images, let's run them in a custom network.

### Step 1: Create a Bridge Network

Create a Docker bridge network named `traffic-lights` with subnet `172.20.0.0/16` and gateway `172.20.0.1`.



<br>

### Step 2: Run the Containers

Run three containers, one for each color, connected to the network, with port mappings: red on 3001, yellow on 3002, green on 3003.

Name them `red-app`, `yellow-app`, `green-app`.


Open each in your browser: `http://localhost:3001`, `http://localhost:3002`, `http://localhost:3003`.

Verify they work.

## Part 3: Using Docker Compose

Managing multiple containers with individual `docker run` commands can be tedious. Docker Compose simplifies this. (TOPIC BREAK)

## Part 4: Nginx Reverse Proxy

### Step 1: Create nginx.conf

Create a file named `nginx.conf` with the following configuration:

```
events {
    worker_connections 1024;
}

http {
    server {
        listen 3000;
        location / {
            proxy_pass http://red-app;
        }
    }

    server {
        listen 4000;
        location / {
            proxy_pass http://yellow-app;
        }
    }

    server {
        listen 5000;
        location / {
            proxy_pass http://green-app;
        }
    }
}
```

### Step 2: Run Nginx Container

Run an Nginx container named `nginx-proxy`, using the `nginx:alpine` image
- map ports 3000, 4000, 5000
- mount the config to `/etc/nginx/nginx.conf`
- connect it to the `traffic-lights` network

Test the endpoints:

- `http://localhost:3000` (red)
- `http://localhost:4000` (yellow)
- `http://localhost:5000` (green)

Also test direct access to the containers (with the previous mapping): `http://localhost:3001`, `3002` etc.

If working, remove the port mappings from the compose file for the light services.

(Update `compose.yaml`)


Restart compose:

```
docker-compose down
docker-compose up -d
```

Now, direct access to 3001-3003 should not work, but through Nginx (3000,4000,5000) should.


## Part 5: Add Nginx to Compose

Migrate the Nginx container to the compose file.

Stop the manual Nginx container:

```
docker stop nginx-proxy
docker rm nginx-proxy
```

Run compose again:

```
docker-compose up -d
```

Verify everything works.

## Part 6: Load Balancing with Nginx

Instead of having separate ports for each color, let's configure Nginx to load balance requests across all three light containers on a single port.

### Step 1: Update nginx.conf

Modify the `nginx.conf` file to use an upstream block for load balancing:

```
events {
    worker_connections 1024;
}

http {
    upstream lights {
        server red-app:80;
        server yellow-app:80;
        server green-app:80;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://lights;
        }
    }
}
```

### Step 2: Update Compose File

In your `compose.yaml`, ensure port 80 is exposed for the Nginx service (e.g., ports: - "80:80"), and remove any other port mappings.

Restart the services:

```
docker-compose down
docker-compose up -d
```

### Step 3: Test Load Balancing

Access `http://localhost:80` multiple times. You should see the page cycling through red, yellow, and green lights as Nginx load balances the requests among the containers.

Verify that all containers are running and accessible through the load balancer. 
