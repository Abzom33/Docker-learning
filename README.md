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

This because I used a multi stage docker-compose.yml in which I did not use the correct dir where the pip packages live : `/usr/local/lib/python3.12/site-packages `. For simplicity I removed the Multi-stage build.

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


New DokcerFile  

```


# Stage 1: Dependencies
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5002
CMD [ "python3" , "app.py" ]


```
