Dockerise your PHP application with Nginx and PHP7-FPM
Before we start…
Before we start, we have to agree on one thing – Docker is super cool! If you are not familiar with Docker, I suggest to have a look at the tons of “Getting starting with Docker” or “What is Docker?” articles and then come back here. :)
Since you keep reading, I will assume that you already have some Docker experience and you want to run your PHP applications in containers. Because who wants the trouble of installing all the dependencies on their local environment  or manage a number of virtual machines for their different projects, right? Right!

The goal that we will try to achieve is to run a simple PHP application using the official Docker repositories for both PHP and Nginx. There are several docker repositories combining PHP-FPM with Nginx, but depending on the official repositories gives you several benefits, like using a service which is configured by its maintainers and you can always choose between the latest and greatest or different versions of both services, instead of relying on someone else’s choices.

The first thing you have to do is, of course, install Docker (if you haven’t already). The second prerequisite is getting Docker Compose (it is included in the Mac toolbox).  Now that we know what we want to achieve and have the tools to accomplish it – let’s get our hands dirty!

Setting up Nginx
We’ll start by getting ourselves a web server and based on our requirements this will be a container running the official Nginx image. Since we’ll be using Docker Compose, we will create the following docker-compose.yml file, which will run the latest Nginx image and will expose its port 80 to port 8080:

web:
 image: nginx:latest
 ports:
 - "8080:80"
Now we can run

docker-compose up
This should give you the default Nginx screen on port 8080 for localhost or the IP of your docker machine.



Now that we have a server let’s add some code. First we have to update the docker-compose.yml to mount a local directory. I will use a folder called code, which is in the same directory as my docker-compose.yml file, and it will be mounted as root folder code in the container.

web:
    image: nginx:latest
    ports:
        - "8080:80"
    volumes:
        - ./code:/code
The next step is to let Nginx know that this folder exists.
Let’s create the following site.conf on the same level as the docker-compose.yml file:

server {
    index index.html;
    server_name php-docker.local;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /code;
}
If you don’t have a lot of experience with Nginx, this is what we define here – index.html will be our default index, the server name is php-docker.local and it should be pointing (update your hosts file) to your Docker environment (localhost if you are on Linux or the docker machine if you are on Mac or Windows), we point the error logs to be the ones exposed by the default container, so that we will see the errors in our docker compose log, and finally we specify the root folder to be the one that we mounted in the container.

To activate this setup we need to apply yet another modification to our docker-compose.yml file:

web:
    image: nginx:latest
    ports:
        - "8080:80"
    volumes:
        - ./code:/code
        - ./site.conf:/etc/nginx/conf.d/site.conf
This will add site.conf to the directory where Nginx is looking for configuration files to include. You can now place an index.html file in the code folder with contents that is to your heart’s delight. And if we run

docker-compose up
again, the index.html file should be available on php-docker.local:8080.

Yeey! We are half way there


Adding PHP-FPM
Now that we have Nginx up and running let’s add the PHP in the game. The first thing we’ll do is pull the official PHP7-FPM repo and link it to our Nginx container. Our docker-compose.yml will look like this now:

web:
    image: nginx:latest
    ports:
        - "8080:80"
    volumes:
        - ./code:/code
        - ./site.conf:/etc/nginx/conf.d/site.conf
    links:
        - php
php:
    image: php:7-fpm
The next thing to do is configure Nginx to use the PHP-FPM container for interpreting PHP files. Your updated site.conf should look like this:

server {
    index index.php index.html;
    server_name php-docker.local;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /code;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
In order to test this let’s rename the index.html file to index.php and replace its content with the standard:

<?php
echo phpinfo();
One final

docker-compose up
And we should be good to go... but

Instead of getting the proper PHP info page we receive the rather unsettling

File not found.
Since PHP is running in its own environment (container) it doesn't have access to the code. In order to fix this, we need to mount the code folder in the PHP container too. This way Nginx will be able to serve any static files, and PHP will be able to find the files it has to interpret. One final change to the docker-compose.yml:

web:
    image: nginx:latest
    ports:
        - "8080:80"
    volumes:
        - ./code:/code
        - ./site.conf:/etc/nginx/conf.d/site.conf
    links:
        - php
php:
    image: php:7-fpm
    volumes:
        - ./code:/code
Finally, this last (this time for real)

docker-compose up
present us with the much wanted PHP info

This is it.

We can run any simple PHP application inside Docker containers, using the official images for Nginx and PHP.

You can find the sample project here https://github.com/mikechernev/dockerised-php

EDIT: Since the GitHub repository changed quite a lot, I added a new blog post explaining the improvements - Making your dockerised PHP application even better
