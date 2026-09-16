## Docker Challenge

Building a Multi Container Application

## Objective

Create a multi-container application that consists of a simple Python Flask web application and a Redis database. The Flask application should use Redis to store and retrieve data.

## Application
These are the Steps I have took

## Steps 1: Setting up Flask application with Redis

1. First create a virtual environment:
        `python -m  venv venv`
        `source venv/bin/activate`
    You should see this if sucessful:
     <img width="338" height="21" alt="image" src="https://github.com/user-attachments/assets/105a8a8d-e49f-4c27-a6c4-e6ccefa790f6" />


2. Install Flask and Redis
  Flask :
  `pip install flask`
  Redis:
  `pip install redis`

2. Next set up requirements.txt file 
- This is where you dependencies is going to be
Commands
`pip freeze > requirements.txt`

To check if requirements.txt has correct dependencies do this :
`cat requirements.txt `

3. Then set up application :

```
 import redis
from flask import Flask

app = Flask(__name__)

r = redis.Redis(host='redis', port=6379)

@app.route('/')
def home():
    return "Welcome to Home Page"
   
@app.route('/count')
def count():
        count = r.incr('visit')
        return f"This page has visited {count} times"
    

if __name__ == "__main__":
    app.run(host='0.0.0.0',port=5002)
```
    
4. Setup DockerFile 
```
FROM python:3.12-slim   --> Base Image
WORKDIR /app      -->  This sets up /app in the image
COPY requirements.txt .    --> Copies requirement.txt from host to image
RUN pip install -r requirements.txt --> Installs the requirements.txt
COPY . .   ---> Copies from the host machine into the image (app.py)
EXPOSE 5002  --> Use port 5002 to run application
CMD [ "python3" , "app.py" ]  -->  First run this command when container starts
```


4a . Build the Docker Image
` docker build -t challenge:v1 .`

5. Setup Docker-compose.yml file

- We use this so we can run both containers within a single command . 
```
version: '3.8'

services:   --> This lists the containers web and redis and ho they should interact
  web:  --> web container
    image: challenge:v1  ---> this refers to the image we built earlier and pulls  the image 
    ports:
      - "5002:5002"
    depends_on:  ---> This will start redis container first
      - redis

  redis:  --> redis container
    image: redis  --> This pulls redis from Docker Hub
    ports:
      - "6379:6379"  --> redis own port

```

## Result :

- http://127.0.0.1:5002/

<img width="1596" height="848" alt="image" src="https://github.com/user-attachments/assets/d5959215-3d72-490d-b3b0-eba4e05d9be9" />




- http://127.0.0.1:5002/count

<img width="1599" height="847" alt="image" src="https://github.com/user-attachments/assets/599da074-4260-4d57-9701-0db5a6b0af62" />




## Challenege
One thing I struggled with was understand where pip packages went. This because when I ran the Docker-compose up when running docker-compose.yml file .

I got this error message : 


<img width="487" height="208" alt="image" src="https://github.com/user-attachments/assets/b972b2d0-7778-4634-a9d6-c99318157f88" />


- This is because the web contianer is unable to find the python redis package , this is beacuse the package sits outside the container.

This was because I used a multi-stage docker-compose.yml in which I did not use the correct dir where the pip packages live: `/usr/local/lib/python3.12/site-packages`. My `COPY --from=build /app /app` line only copied the `/app` directory from the build stage, not the installed packages themselves. pip installs into that site-packages path, which lives outside `/app`, so none of it made it into the final image. For simplicity I removed the multi-stage build since it is not necessary for this challenge.

Multi-stage build :

this is my dokcerfile 
```
 Stage 1: Dependencies
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
RUN pip install flask redis


 Stage 2 : Run application

FROM python:3.12-slim 
WORKDIR /app
COPY --from=build /app /app
EXPOSE 5002
CMD [ "python3" , "app.py" ]

```

## Bonus

## Objective

The following objective is :
- Persistent Storage for Redis: Configure Redis to use a volume to persist its data.
- Environment Variables: Modify the Flask application to read Redis connection details from environment variables and update the docker-compose.yml accordingly.

## Persistent Storage
So far , in this challenge our data gets overwritten every time when we rerun our container. Therefore we lose data. To combat this we use something called  `volumes` in our docker-compose.yml file.

`volumes` - It is a persistent storage that stores your data outside your container . Even if the container is shutdown still keeps and stores data.

You need to declare it inside your docker-compose.yml
```
volumes:
  db_data:   ----> named volumes
```
Also you need to declare explicity inside your services

```
redis:
    image: redis
    volumes:
    - db_data:/data   ---->   named volume:/path
  
```
redis stores its data at `/data`


## Errors

- `services.volumes must be a mapping`
  This was due to a syntax error
  Ans : Indentation Error


`validating /home/qalay/Docker Challenge/docker-compose.yml: services.volumes additional properties 'db_data' not allowed`

This happend as I have identended the global variable volumes under the services
Ans : Put the volumes variable  on the same ident as services

`Error: /home/qalay/Docker Challenge/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion `
service "redis" refers to undefined volume db_data: invalid compose project

- This is because it does not know db_data is since the named volume is db-data so I need to change it to make it match



## Environment Variables: Modify the Flask application to read Redis connection details from environment variables and update the docker-compose.yml accordingly.

Environment variable - This is a dynamic key pair value that is used configure applications without being hardcoded

To do this in Docker-compose.yml file : 


```
environment:
    key=value
```
To modify flask application to read the env variable we need to import OS libary and we need to use getenv variable

`os.getenv(key , default)` --> This is used to grab environment variable .

- `key` --> name of env variable to look up
- `default` ---> The value to look for incase it can't find env variable


```
redis_host = os.getenv('host', 'localhost')
redis_port=os.getenv('port', 6379)
r = redis.Redis(host=redis_host, port=redis_port)

```


```
  web:
    image: challenge:v1
    volumes:
    - db_data:/data
      
    ports:
      - "5002:5002"
    depends_on:
      - redis
    environment:
      - host=redis
      - port=6379



```

## Scaling the Application: Scale the Flask service to run multiple instances and load balance between them using Docker Compose.

For this section , we are presented with a task to scale our flask service and load balance between each instance.
Before we start lets define what is load balancing ?

## Load balancing 
-  It is the process of spreading network traffic across different servers to keep the application fast and reliable
- For this section we are going to use `NGINX` to be our load balancer.

## Steps

1. First create a `nginx.conf file` . This file will be responsible for load balancing between different server.


```
events {}

http {
    # Define the group of servers available
    upstream app {
        server web:5002;

    }
    server {
        # Server group will respond to port 5002
        listen 5002;
        server_name web;
        location / {
            proxy_pass http://app;
        }
    }
}
```
2. Update the docker-compose.yml file by creating nginx service that will mount to our nginx.conf file that will be able to apply load balancing.
   

```
nginx:
    image: nginx:latest
    
    volumes:
        - ./nginx.conf:/etc/nginx/nginx.conf
    
    ports:
      - 5002:5002  ---> the `web` service should not  be mapped to 5002 host to conatiner port as this will cause a port conflict. Only nginx should be so it can acesss our         container

    depends_on:
      - web

```




3. Scale our flask application

- Usr this command to do so :

`docker-compose up -d --scale name=scale`

For example if we want to scale our web application to 3 different servers

`docker-compose up -d --scale web=3`






