# Development Sql Setup
We want a MySQL instance running on our personal development machine which is Windows 10, but we need the data stored on another drive other than C:\\.  Note that these steps won't take place in the remote (prod and test) SQL instances, because those will use managed hosting in the cloud.

## Goals:
 * Host MySQL in a docker container under Windows 10.
 * Specify an alternative data storage location for SQL.  This is important for my setup, since I have a large amount of data, and my default OS disk is an SSD, and I need the data stored on a plain HDD that is a different disk.
 * Easily run a command line mysql client in a standard way.

It turns out the combination of running the container under Windows 10 and specifying an alternative storage location took a good amount of research and experimentation.  Here's how I did it.

## Overview of Steps:
1. **Volumes**.  File storage in the hosting machine that get shared and exported into the container.
2. **Configuration**. Config files for mysql that enable you to connect.
3. **Docker compose**.
4. **Operations**. Starting, stopping, and connecting to the sql instance as a client.

# 1. Volumes
*Volumes* in Docker are sections of the filesystem on the host machine that get shared into the container.  So whenever the container reads and writes to some directory the files are visible to the host OS in a defined location.  We'll be using two volumes: one for data storage, and one for configuration.

In my case I've got my data stored in drive D:.

**Create a folder structure on your Windows drive like this:**
```
    D:\
    └── local\
        └── sql\
            └── rico\
                └── data\
                └── conf\
```
Purpose of each of these folders:
* **D:\local** Where I keep large data for long-term storage.
* **D:\local\sql** Where I keep sql data.
* **D:\local\sql\rico** "Rico" is the name of the sql data I'm using for this sample. 
* **D:\local\sql\rico\data** The sql data storage for the "Rico" sql instance.
* **D:\local\sql\rico\conf** The configuration directory. It will contain a *my.cnf* for the server and client.

# 2. Configuration
Create **D:\local\sql\rico\conf\my.cnf** with this:
```
[mysqld]
socket=/tmp/mysql-rico-sock

[client]
socket=/tmp/mysql-rico-sock
```
The name of the socket can be whatever you want, but the important thing is that it *cannot* be in the default location where MySQL usually puts it.  The default socket location is in /var/lib/mysql, which we're overriding with a separate volume.  When hosting a MySQL container in Windows, you can't put the socket in a filesystem that's working in a volume. References:
    [\[1\]](https://stackoverflow.com/questions/48731913/cant-start-server-bind-on-unix-socket-operation-not-permitted)
    [\[2\]](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_socket)
    [\[3\]](https://dev.mysql.com/doc/refman/8.0/en/problems-with-mysql-sock.html).

# 3. Docker-compose
**Create a docker-compose.yml file like this:**
```yml
version: '3'

services:
  db:
    image: mysql/mysql-server:8.0.13
    container_name: mysql-rico
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'put_something_here'
    volumes:
      - D:\local\sql\rico1\data:/var/lib/mysql
      - D:\local\sql\rico1\conf:/etc/mysql
```
Notes about changes you might want to make:

* The ```container_name``` section specifies the name of this service, which we'll use in later docker commands that you'll see in the "Operations" section below.

* I used a specific tag there on image, because I don't want my database upgrading without me thinking about it explicitly.  You might want to check what the latest container tag is and lock it in there.  8.0.13 is the latest as of the time of this writing.  You can also use ```:latest``` to pull in the latest, but note that it will replace the image with the latest whenever you do a build if you roll that way.

* Depending on your situation, [you may want this in there too](https://mysqlserverteam.com/upgrading-to-mysql-8-0-default-authentication-plugin-considerations/), but I didn't use it:
```yml
command: --default-authentication-plugin=mysql_native_password
```


# 5. Operations
* **Starting Server**
    * Bring the server up, with ^C to stop it:
    ```
    docker-compose up --build
    ```
    The ```--build``` option is there so that it recognizes changes you've made to your **docker-compose.yml** file.  You won't need the --build there normally.

    * Bring the server up and leave it up:
    ```
    docker-compose up --detach
    ```

    * Stop the server
    ```
    docker-compose down
    ```

* **Connecting from the command line**
    Connect a shell to the docker instance.
    ```
    $ winpty docker exec -ti mysql-rico mysql -uroot -p
    ```
    Notice the ```winpty``` command at the beginning.  This makes interactive shells work from git bash and similar shells based on mingw.  You might not need that depending on how you're connecting.  The normal command just starts with ```docker```.

* **Starting a service composed of multiple compose files**
The above **docker-compose.yml** file will actually be just one part of several severices that make up our application.  To start multiple services at once do something like this:
    ```
    docker-compose -f docker-compose.common.yml -f docker-compose.dev.yml up
    ```
    That launches multiple docker-compose files at once.
    
    For a lot more ideas about splitting your application into multiple sets of files, some of which are just for your local environment, and others which are only used in test and prod, see this awesome HackerNoon post about [Efficient Development with Docker and Docker-Compose](https://hackernoon.com/efficient-development-with-docker-and-docker-compose-e354b4d24831).

# References
* [MySQL Reference Manual: Problems with MySQL Socket](https://dev.mysql.com/doc/refman/8.0/en/problems-with-mysql-sock.html)
* docker-compose.yml Based on [Docker Documentation: Compose and Django](https://docs.docker.com/compose/django/)
* [Separating development environment from test and prod](https://testdriven.io/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)
* [Mysql Docker Reference](https://hub.docker.com/_/mysql)
* [Stack overflow: permission denied docker windows mysql](https://stackoverflow.com/questions/48731913/cant-start-server-bind-on-unix-socket-operation-not-permitted)
* [Mysql Manual: nonlinux docker](https://dev.mysql.com/doc/refman/5.7/en/deploy-mysql-nonlinux-docker.html)
* [Mysql Manual: --socket option](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_socket)
* [Configuring Docker MySQL](https://blog.pythian.com/configuring-mysql-docker-container/)
* [ChloeCodesThings Flask/Docker demo](https://github.com/ChloeCodesThings/chloe_flask_docker_demo)