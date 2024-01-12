# Lab 2 : Add CI/CD value with Docker images

Sourced from : https://apps.cognitiveclass.ai/learning/course/course-v1:IBMDeveloperSkillsNetwork+CO0101EN+v1/home

Edited by G.Blanch

&nbsp;    
&nbsp;  


## Introduction 

In this lab, build on your knowledge from Lab 1 where you used Docker commands to run containers. Create a custom Docker image built from a Dockerfile. After you build the image, you will push it to a central registry where it can be pulled to be deployed on other environments.

Also, you'll get a brief understanding of image layers and how Docker incorporates copy-on-write and the union file system to efficiently store images and run containers. You will use a few Docker commands in this lab. See the documentation on for information on available commands.

Remember these key points:

- Use the **Dockerfile** to create reproducible builds for your application and to *integrate* your application with Docker *into the CI/CD pipeline.*
- Docker images can be made *available to all of your environments* through a **central registry**. The *Docker Hub* is one *example of a registry*, but *you can deploy your own registry on servers you control*.
- A Docker image contains all the dependencies that it needs to run an application within the image. This is useful because you *no longer need to deal with environment drift* (version differences) where you rely on dependencies that are installed on every environment you deploy to.
- Docker uses of the *union file system and **"copy-on-write"*** to reuse layers of images. This lowers the footprint of storing images and significantly increases the performance of starting containers.
- *Image layers are cached by the **Docker build and push system***. There's no need to rebuild or repush image layers that are already present on a system.
- Each line in a Dockerfile creates a new layer, and because of the layer cache, the *lines that change more frequently*, for example, adding source code to an image, **should be listed near the bottom of the file**.



## Index

[Create a Python app (without using Docker)](#create-a-python-app-(without-using-docker))

[Create and build the Docker image](#create-and-build-the-docker-image)

[Run the Docker image](#run-the-docker-image)

[Deploy a change](#deploy-a-change)

[Understand image layers](#understand-image-layers)

[Remove containers](#remove-containers)


### Create a Python app (without using Docker)

1. Copy and paste this entire command into the terminal. The result of running this command will create a file named app.py.

```python
echo 'from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "hello world!"

if __name__ == "__main__":
    app.run(host="0.0.0.0")' > app.py
```

This is a simple Python app that uses Flask to expose an HTTP web server on port 5000. (5000 is the default port for flask.) Don't worry if you are not too familiar with Python or Flask. These concepts can be applied to an application written in any language.

2. Optional: If you have Python and pip installed, run this app locally. If not, move on to the next section of this lab.

        $ python3 --version
        Python 3.6.1
        $ pip3 --version
        pip 9.0.1 from /usr/local/lib/python3.6/site-packages (python 3.6)
        $ pip3 install flask
        Requirement already satisfied: flask in /usr/local/lib/python3.6/site-packages
        Requirement already satisfied: Werkzeug>=0.7 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: itsdangerous>=0.21 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: Jinja2>=2.4 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: click>=2.0 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: MarkupSafe>=0.23 in /usr/local/lib/python3.6/site-packages (from Jinja2>=2.4->flask)
        johns-mbp:test johnzaccone$ pip3 install flask
        Requirement already satisfied: flask in /usr/local/lib/python3.6/site-packages
        Requirement already satisfied: itsdangerous>=0.21 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: Jinja2>=2.4 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: click>=2.0 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: Werkzeug>=0.7 in /usr/local/lib/python3.6/site-packages (from flask)
        Requirement already satisfied: MarkupSafe>=0.23 in /usr/local/lib/python3.6/site-packages (from Jinja2>=2.4->flask)
        $ python3 app.py 
        * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)


[Back to Index](#Index)
&nbsp;    
&nbsp;  

### Create and build the Docker image

If you don't have Python installed locally, don't worry because you don't need it. One of the advantages of using Docker containers is that you can build Python into your containers without having Python installed on your host.

Let's do a simple example using **NGINX**. For now, you will create a service with one running container, but you will scale up later.

1.  Create a file named Dockerfile using the touch command. This will create an empty Dockerfile. A Dockerfile is basically a text document that contains all the commands a user could call on the command line to assemble an image.

            touch Dockerfile

2. Add the following content to the `Dockerfile`:

            FROM python:3.6.1-alpine
            RUN pip install --upgrade pip
            RUN pip install flask
            CMD ["python","app.py"]
            COPY app.py /app.py

    Let's understand the commands in the Dockerfile line by line:

    - **FROM python:3.6.1-alpine**

        This is the starting point for your Dockerfile. Every Dockerfile typically starts with a FROM line that is the starting image to build your layers on top of. In this case, you are selecting the python:3.6.1-alpine base layer because it already has the version of Python and pip that you need to run your application. The alpine version means that it uses the alpine distribution, which is significantly smaller than an alternative flavor of Linux. A smaller image means it will download (deploy) much faster, and it is also more secure because it has a smaller attack surface.

        Here you are using the 3.6.1-alpine tag for the Python image. Look at the available tags for the official Python image on the Docker Hub. It is best practice to use a specific tag when inheriting a parent image so that changes to the parent dependency are controlled. If no tag is specified, the latest tag takes effect, which acts as a dynamic pointer that points to the latest version of an image.

        For security reasons, you must understand the layers that you build your docker image on top of. For that reason, it is highly recommended to only use official images found in the Docker Hub, or noncommunity images found in the Docker Store. These images are vetted to meet certain security requirements, and also have very good documentation for users to follow. You can find more information about this Python base image and other images that you can use on the Docker store.

        For a more complex application, you might need to use a FROM image that is higher up the chain. For example, the parent Dockerfile for your Python application starts with FROM alpine, then specifies a series of CMD and RUN commands for the image. If you needed more control, you could start with FROM alpine (or a different distribution) and run those steps yourself. However, to start, it's recommended that you use an official image that closely matches your needs.

    - **RUN pip install flask**

        The RUN command executes commands needed to set up your image for your application, such as installing packages, editing files, or changing file permissions. In this case, you are installing Flask. The RUN commands are executed at build time and are added to the layers of your image.

    - **CMD ["python","app.py"]**

        CMD is the command that is executed when you start a container. Here, you are using CMD to run your Python application.

        There can be only one CMD per Dockerfile. If you specify more than one CMD, then the last CMD will take effect. The parent python:3.6.1-alpine also specifies a CMD (CMD python2). You can look at the Dockerfile for the official python:alpine image.

        You can use the official Python image directly to run Python scripts without installing Python on your host. However, in this case, you are creating a custom image to include your source so that you can build an image with your application and ship it to other environments.

    - **COPY app.py /app.py**

        This line copies the app.py file in the local directory (where you will run docker image build) into a new layer of the image. This instruction is the last line in the Dockerfile. Layers that change frequently, such as copying source code into the image, should be placed near the bottom of the file to take full advantage of the Docker layer cache. This allows you to avoid rebuilding layers that could otherwise be cached. For instance, if there was a change in the FROM instruction, it will invalidate the cache for all subsequent layers of this image. You'll see this little later in this lab.

        It seems counter-intuitive to put this line after the CMD ["python","app.py"] line. Remember, the CMD line is executed only when the container is started, so you won't get a file not found error here.

        And there you have it: a very simple Dockerfile. See the full list of commands that you can put into a Dockerfile. Now that you've defined the Dockerfile, you'll use it to build your custom docker image.

3. Return to terminal  window and type the command given below. You can verify the contents of the Dockerfile within the terminal window.

            vi Dockerfile

4. Now enter the command below  and press <Enter>  to close the Dockerfile opened within the terminal

            :wq

5. Now that you've defined the Dockerfile, you'll use it to build your custom docker image. Pass in the -t parameter to name your image python-hello-world:

            $ docker image build -t python-hello-world .
            Sending build context to Docker daemon  3.072kB
            Step 1/4 : FROM python:3.6.1-alpine
            3.6.1-alpine: Pulling from library/python
            acb474fa8956: Pull complete 
            967ab02d1ea4: Pull complete 
            640064d26350: Pull complete 
            db0225fcac8f: Pull complete 
            5432cc692c60: Pull complete 
            Digest: sha256:768360b3fad01adffcf5ad9eccb4aa3ccc83bb0ed341bbdc45951e89335082ce
            Status: Downloaded newer image for python:3.6.1-alpine
            ---> c86415c03c37
            Step 2/4 : RUN pip install flask
            ---> Running in cac3222673a3
            Collecting flask
            Downloading Flask-0.12.2-py2.py3-none-any.whl (83kB)
            Collecting itsdangerous>=0.21 (from flask)
            Downloading itsdangerous-0.24.tar.gz (46kB)
            Collecting click>=2.0 (from flask)
            Downloading click-6.7-py2.py3-none-any.whl (71kB)
            Collecting Werkzeug>=0.7 (from flask)
            Downloading Werkzeug-0.12.2-py2.py3-none-any.whl (312kB)
            Collecting Jinja2>=2.4 (from flask)
            Downloading Jinja2-2.9.6-py2.py3-none-any.whl (340kB)
            Collecting MarkupSafe>=0.23 (from Jinja2>=2.4->flask)
            Downloading MarkupSafe-1.0.tar.gz
            Building wheels for collected packages: itsdangerous, MarkupSafe
            Running setup.py bdist_wheel for itsdangerous: started
            Running setup.py bdist_wheel for itsdangerous: finished with status 'done'
            Stored in directory: /root/.cache/pip/wheels/fc/a8/66/24d655233c757e178d45dea2de22a04c6d92766abfb741129a
            Running setup.py bdist_wheel for MarkupSafe: started
            Running setup.py bdist_wheel for MarkupSafe: finished with status 'done'
            Stored in directory: /root/.cache/pip/wheels/88/a7/30/e39a54a87bcbe25308fa3ca64e8ddc75d9b3e5afa21ee32d57
            Successfully built itsdangerous MarkupSafe
            Installing collected packages: itsdangerous, click, Werkzeug, MarkupSafe, Jinja2, flask
            Successfully installed Jinja2-2.9.6 MarkupSafe-1.0 Werkzeug-0.12.2 click-6.7 flask-0.12.2 itsdangerous-0.24
            ---> ce41f2517c16
            Removing intermediate container cac3222673a3
            Step 3/4 : CMD python app.py
            ---> Running in 2197e5263eff
            ---> 0ab91286958b
            Removing intermediate container 2197e5263eff
            Step 4/4 : COPY app.py /app.py
            ---> f1b2781b3111
            Removing intermediate container b92b506ee093
            Successfully built f1b2781b3111
            Successfully tagged python-hello-world:latest

8. Verify that your image shows in your image list:

            $ docker image ls

            REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
            python-hello-world   latest              f1b2781b3111        26 seconds ago      99.3MB
            python               3.6.1-alpine        c86415c03c37        8 days ago          88.7MB

Notice that your base image, python:3.6.1-alpine, is also in your list


[Back to Index](#Index)
&nbsp;    
&nbsp;  


### Run the Docker image


Now that you have built the image, you can run it to see that it works.

1. Run the Docker image:

            $ docker run -p 5001:5000 -d python-hello-world
            0b2ba61df37fb4038d9ae5d145740c63c2c211ae2729fc27dc01b82b5aaafa26

    The `-p` flag maps a port running inside the container to your host. In this case, you're mapping the Python app running on port 5000 inside the container to port 5001 on your host. Note that if port 5001 is already being used by another application on your host, you might need to replace 5001 with another value, such as 5002.

2. Navigate to http://localhost:5001 in a browser to see the results.

<img align="center" src="">


3. Check the log output of the container.

    If you want to see logs from your application, you can use the docker container logs command. By default, docker container logs prints out what is sent to standard out by your application. Use the command docker container ls to find the ID for your running container.

            $ docker container logs [container id] 
            * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
            172.17.0.1 - - [28/Jun/2017 19:35:33] "GET / HTTP/1.1" 200 -

    The Dockerfile is used to create reproducible builds for your application. A common workflow is to have your CI/CD automation run docker image build as part of its build process. After images are built, they will be sent to a central registry where they can be accessed by all environments (such as a test environment) that need to run instances of that application. In the next section, you will push your custom image to the public Docker registry, which is the Docker Hub, where it can be consumed by other developers and operators.

[Back to Index](#Index)
&nbsp;    
&nbsp;  

### Push to a central registry


1. Navigate to Docker Hub and create a free account if you haven't already.

    For this lab, you will use the Docker Hub as your central registry. Docker Hub is a free service to publicly store available images. You can also pay to store private images.

    Most organizations that use Docker extensively will set up their own registry internally. To simplify things, you will use  Docker Hub, but the following concepts apply to any registry.

2. Log in to the Docker registry account by entering docker login on your terminal:

            $ docker login
            Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
            Username: 

3. Tag the image with your username.

    The Docker Hub naming convention is to tag your image with ```[dockerhub username]/[image name]```. To do this, tag your previously created image python-hello-world to fit that format.

            $ docker tag python-hello-world [dockerhub username]/python-hello-world

4. After you properly tag the image, use the docker push command to push your image to the Docker Hub registry:

            $ docker push jzaccone/python-hello-world
            The push refers to a repository [docker.io/jzaccone/python-hello-world]
            2bce026769ac: Pushed 
            64d445ecbe93: Pushed 
            18b27eac38a1: Mounted from library/python 
            3f6f25cd8b1e: Mounted from library/python 
            b7af9d602a0f: Mounted from library/python 
            ed06208397d5: Mounted from library/python 
            5accac14015f: Mounted from library/python 
            latest: digest: sha256:508238f264616bf7bf962019d1a3826f8487ed6a48b80bf41fd3996c7175fd0f size: 1786

5. Check your image on Docker Hub in your browser.

    Navigate to Docker Hub and go to your profile to see your uploaded image.

    Now that your image is on Docker Hub, other developers and operators can use the docker pull command to deploy your image to other environments.

    **Remember**: Docker images contain all the dependencies that they need to run an application within the image. This is useful because you no longer need to worry about environment drift (version differences) when you rely on dependencies that are installed on every environment you deploy to. You also don't need to follow more steps to provision these environments.

[Back to Index](#Index)
&nbsp;    
&nbsp;  

### Deploy a change

1. Update app.py by replacing the string "Hello World" with "Hello Beautiful World!" in app.py as follows:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello Beautiful World!"


if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

Now that your application is updated, you need to rebuild your app and push it to the Docker Hub registry.

2. Rebuild the app by using your Docker Hub username in the build command:

            $  docker image build -t jzaccone/python-hello-world .
            Sending build context to Docker daemon  3.072kB
            Step 1/4 : FROM python:3.6.1-alpine
            ---> c86415c03c37
            Step 2/4 : RUN pip install flask
            ---> Using cache
            ---> ce41f2517c16
            Step 3/4 : CMD python app.py
            ---> Using cache
            ---> 0ab91286958b
            Step 4/4 : COPY app.py /app.py
            ---> 3e08b2eeace1
            Removing intermediate container 23a955e881fc
            Successfully built 3e08b2eeace1
            Successfully tagged jzaccone/python-hello-world:latest

Notice the "Using cache" for Steps 1 - 3. These layers of the Docker image have already been built, and the docker image build command will use these layers from the cache instead of rebuilding them.

            $ docker push jzaccone/python-hello-world
            The push refers to a repository [docker.io/jzaccone/python-hello-world]
            94525867566e: Pushed 
            64d445ecbe93: Layer already exists 
            18b27eac38a1: Layer already exists 
            3f6f25cd8b1e: Layer already exists 
            b7af9d602a0f: Layer already exists 
            ed06208397d5: Layer already exists 
            5accac14015f: Layer already exists 
            latest: digest: sha256:91874e88c14f217b4cab1dd5510da307bf7d9364bd39860c9cc8688573ab1a3a size: 1786
            There is a caching mechanism in place for pushing layers too. Docker Hub already has all but one of the layers from an earlier push, so it only pushes the one layer that has changed.

When you change a layer, every layer built on top of that will have to be rebuilt. Each line in a Dockerfile builds a new layer that is built on the layer created from the lines before it. This is why the order of the lines in your Dockerfile is important. You optimized your Dockerfile so that <ins>the layer that is most likely to change (COPY app.py /app.py) is the last line of the Dockerfile.</ins> Generally for an application, your code changes at the most frequent rate. This optimization is particularly important for CI/CD processes where you want your automation to run as fast as possible.


### Understand image layers

One of the important design properties of Docker is its use of the union file system.

Consider the Dockerfile that you created before:

            FROM python:3.6.1-alpine
            RUN pip install flask
            CMD ["python","app.py"]
            COPY app.py /app.py

Each of these lines is a layer. Each layer contains only the *delta, or changes from the layers before it.* To put these layers together into a single running container, Docker uses the union file system to overlay layers transparently into a single view.

Each layer of the image is read-only except for the top layer, which is created for the container. The read/write container layer implements "copy-on-write," which means that files that are stored in lower image layers are pulled up to the read/write container layer only when edits are being made to those files. Those changes are then stored in the container layer.

The "copy-on-write" function is very fast and in almost all cases, does not have a noticeable effect on performance. You can inspect which files have been pulled up to the container level with the docker diff command. For more information, see the command-line reference on the ```docker diff``` command.


<p align="center">
<img src="https://github.com/GBlanch/Docker-Essentials-Developer-Introduction/blob/main/Lab%202%20-%20Add%20CICD%20value%20with%20Docker%20images/assets/rw%20and%20ils.png" width="50%" height="50%">


Because image layers are read-only, *they can be shared by images and by running containers.* For example, creating a new Python application with its own Dockerfile with similar base layers will share all the layers that it had in common with the first Python application.

            FROM python:3.6.1-alpine
            RUN pip install flask
            CMD ["python","app2.py"]
            COPY app2.py /app2.py


<p align="center">
<img src="https://github.com/GBlanch/Docker-Essentials-Developer-Introduction/blob/main/Lab%202%20-%20Add%20CICD%20value%20with%20Docker%20images/assets/unique%20and%20shared%20ls.png" width="50%" height="50%">



You can also see the sharing of layers when you start multiple containers from the same image. Because the containers use the same read-only layers, you can imagine that starting containers is very fast and has a very low footprint on the host.

You might notice that there are duplicate lines in this Dockerfile and the Dockerfile that you created earlier in this lab. Although this is a trivial example, you can pull common lines of both Dockerfiles into a base Dockerfile, which you can then point to with each of your child Dockerfiles by using the FROM command.

Image layering enables the docker caching mechanism for builds and pushes. For example, the output for your last docker push shows that some of the layers of your image already exist on the Docker Hub.

            $ docker push jzaccone/python-hello-world
            The push refers to a repository [docker.io/jzaccone/python-hello-world]
            94525867566e: Pushed 
            64d445ecbe93: Layer already exists 
            18b27eac38a1: Layer already exists 
            3f6f25cd8b1e: Layer already exists 
            b7af9d602a0f: Layer already exists 
            ed06208397d5: Layer already exists 
            5accac14015f: Layer already exists 
            latest: digest: sha256:91874e88c14f217b4cab1dd5510da307bf7d9364bd39860c9cc8688573ab1a3a size: 1786

To look more closely at layers, you can use the docker image history command of the Python image you created.

            $ docker image history python-hello-world
            IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
            f1b2781b3111        5 minutes ago       /bin/sh -c #(nop) COPY file:0114358808a1bb...   159B                
            0ab91286958b        5 minutes ago       /bin/sh -c #(nop)  CMD ["python" "app.py"]      0B                  
            ce41f2517c16        5 minutes ago       /bin/sh -c pip install flask                    10.6MB              
            c86415c03c37        8 days ago          /bin/sh -c #(nop)  CMD ["python3"]              0B                  
            <missing>           8 days ago          /bin/sh -c set -ex;   apk add --no-cache -...   5.73MB              
            <missing>           8 days ago          /bin/sh -c #(nop)  ENV PYTHON_PIP_VERSION=...   0B                  
            <missing>           8 days ago          /bin/sh -c cd /usr/local/bin  && ln -s idl...   32B                 
            <missing>           8 days ago          /bin/sh -c set -ex  && apk add --no-cache ...   77.5MB              
            <missing>           8 days ago          /bin/sh -c #(nop)  ENV PYTHON_VERSION=3.6.1     0B                  
            <missing>           8 days ago          /bin/sh -c #(nop)  ENV GPG_KEY=0D96DF4D411...   0B                  
            <missing>           8 days ago          /bin/sh -c apk add --no-cache ca-certificates   618kB               
            <missing>           8 days ago          /bin/sh -c #(nop)  ENV LANG=C.UTF-8             0B                  
            <missing>           8 days ago          /bin/sh -c #(nop)  ENV PATH=/usr/local/bin...   0B                  
            <missing>           9 days ago          /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B                  
            <missing>           9 days ago          /bin/sh -c #(nop) ADD file:cf1b74f7af8abcf...   4.81MB  

Each line represents a layer of the image. You'll notice that the top lines match to the Dockerfile that you created, and the lines below are pulled from the parent Python image. Don't worry about the <missing> tags. These are still normal layers; they have just not been given an ID by the Docker system.

### Remove the containers

Completing this lab results in a lot of running containers on your host. You'll stop and remove these containers.

1. Get a list of the containers running by running the command docker container ls:

            $ docker container ls
            CONTAINER ID        IMAGE                COMMAND             CREATED             STATUS              PORTS                    NAMES
            0b2ba61df37f        python-hello-world   "python app.py"     7 minutes ago       Up 7 minutes        0.0.0.0:5001->5000/tcp   practical_kirch

2. Run `docker container stop [container id]` for each container in the list that is running:

            $ docker container stop 0b2
            0b2

3. Remove the stopped containers by running docker system prune:

            $ docker system prune
            WARNING! This will remove:
                    - all stopped containers
                    - all volumes not used by at least one container
                    - all networks not used by at least one container
                    - all dangling images
            Are you sure you want to continue? [y/N] y
            Deleted Containers:
            0b2ba61df37fb4038d9ae5d145740c63c2c211ae2729fc27dc01b82b5aaafa26

            Total reclaimed space: 300.3kB
