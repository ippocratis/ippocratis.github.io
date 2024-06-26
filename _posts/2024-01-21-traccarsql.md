---
layout: post
title:  "Traccar location tracker"
author: ippo
categories: [ map, location ]
tags: [ map, location ]
image: assets/images/traccar.png
---

Updated the guide with a mysql dB as postgess was not really working . Also re-phrased the google takeout migration to make more sense.

Get location history with a selfhosted dockerised location tracking server and an android client to provide periodic location updates

**create a proper Dockerfile for raspberry aarch**

`nano Dockerfile`

paste

```
FROM debian:latest

ENV TRACCAR_VERSION 4.10

WORKDIR /opt/traccar

RUN set -ex && \
    apt-get update &&\
    TERM=xterm DEBIAN_FRONTEND=noninteractive apt-get install --yes --no-install-recommends unzip wget default-jre && \
    wget -qO /tmp/traccar.zip https://github.com/traccar/traccar/releases/download/v$TRACCAR_VERSION/traccar-other-$TRACCAR_VERSION.zip && \
    unzip -qo /tmp/traccar.zip -d /opt/traccar && \
    apt-get autoremove --yes unzip wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/*

ENTRYPOINT ["java", "-Xms512m", "-Xmx512m", "-Djava.net.preferIPv4Stack=true"]

CMD ["-jar", "tracker-server.jar", "conf/traccar.xml"]
```

**build the Dockerfile**

`docker build -t debian-traccar-nginx .`

**create the volume**

`docker volume create traccar_raspberry`

**run the container**

```
docker run -d \
  --name Traccar \
  -p 5002:5002/tcp \
  -p 5002:5002/udp \
  -p 5055:5055/tcp \
  -p 5093:5093/tcp \
  -p 5093:5093/udp \
  -p 82:8082 \
  --mount source=traccar_raspberry,target=/opt/traccar \
  --restart=unless-stopped \
  debian-traccar-nginx:latest
```

update: the above Dockerfile is no longer needed
tested the debian atm64 image and it works fine on raspberry400
Digest:sha256:f09791c90dbfe9ea5adeb5c5c1f657cfdb1eccaf56bda2194f2a84abfb4b6412

https://hub.docker.com/layers/traccar/traccar/debian/images/sha256-f09791c90dbfe9ea5adeb5c5c1f657cfdb1eccaf56bda2194f2a84abfb4b6412?context=explore

android client needs tcp 5055 port so if you dont plan to use any of the supported hardware gps trackers
you can skip listening to 5002,5093 tcp/udp ports and only keep 5055 for events and 8082 for the webui

**docker-compose file**

*assuming you have an external hard drive mounted as TOSHIBA  on  /run/media/ippo/TOSHIBA and your username is ippo

create dirs 
that will have all are files and db on our external drive for backup reasons

mkdir -p /run/media/ippo/TOSHIBA/traccar
                         
mkdir -p /run/media/ippo/TOSHIBA/traccar/logs
                  
mkdir -p /run/media/ippo/TOSHIBA/traccar/data

mkdirn-p /run/media/ippo/TOSHIBA/traccar/conf

**get the trackar.xml configuration file from the builded image**

`docker run --rm --entrypoint cat debian-traccar-nginx:latest /opt/traccar/conf/traccar.xml > /run/media/ippo/TOSHIBA/traccar/conf/traccar.xml`

**the actual yml**

```
docker-compose.yml

version: '2'
services:
    web:
        image: 'debian-traccar-nginx:latest'
        restart: unless-stopped
        hostname: 'my.ddns.provider.com'
        container_name: traccar
        ports:
            - '8082:8082'
            - '5055:5055/tcp'
        volumes:
            - '/etc/localtime:/etc/localtime:ro'
            - '/run/media/ippo/TOSHIBA/traccar/logs:/opt/traccar/logs:rw'
            - '/run/media/ippo/TOSHIBA/traccar/conf/traccar.xml:/opt/traccar/conf/traccar.xml:ro'
            - '/run/media/ippo/TOSHIBA/traccar/data:/opt/traccar/data:rw'
```

**run the container**

`doker-cmpose up -d`

the above will spinup traccar with an  embedded h2 db that will not reset on container restart as its
volume is mounted  on host and take periodic copies of the files but once corrupted those copies cant restore the traccar container
Also I have no idea  how to properly backup an h2 db (if possible at all)
the recomended way is to use an external postges or mariadb database
i used a postgres docker imageb connected to the traccar image with a docker network and provided the apropriate network gateway url to the traccar xml for the database url

so:

**traccar-mysql**


Replace the h2 contents in the xml with the postges according to
https://www.traccar.org/mysql/


docker-compose.yml

```yml
 version: "3"

services:
    traccar-db:
        image: yobasystems/alpine-mariadb
        container_name: traccar-db
        command: --default-authentication-plugin=mysql_native_password
        restart: always
        volumes:
            - /run/media/ippo/TOSHIBA/traccar/mysql-data:/var/lib/mysql
            - /run/media/ippo/TOSHIBA/traccar/mysql:/etc/mysql/conf.d
        ports:
            - "3306:3306"
        environment:
            - MYSQL_ROOT_PASSWORD=xxx
            - MYSQL_DATABASE=traccar-db
            - MYSQL_USER=xxx
            - MYSQL_PASSWORD=xxx
        networks:
            - iot
            
    traccar:
        image: traccar/traccar:debian
        hostname: XXX.XXX.com
        container_name: traccar
        depends_on:
           - traccar-db
        restart: always
        volumes:
            - /run/media/ippo/TOSHIBA/traccar/conf/traccar.xml:/opt/traccar/conf/traccar.xml:ro
            - /run/media/ippo/TOSHIBA/traccar/logs:/opt/traccar/logs:rw
        ports:
            - "5055:5055"
            - "82:8082"
        networks:
            - iot
            
networks:
  iot:
    driver: bridge
    enable_ipv6: false
    ipam:
      config:
        - subnet: 10.0.1.0/29
 
```


`docker network inspect iot`

under "Name": "iot",we need "Subnet": "10.0.1.0/29"

and under "Iot",
we need "IPv4Address": " 10.0.1.2",

subnet goes to 

 config:
        - subnet:  10.0.1.0/29
in docker-compose

and

IPv4Address goes to jdbc:mysql://docker-bridge-ipv4:container-internal-port/database-name so for our example it will be : jdbc:mysql:// 10.0.1.2:3306/traccar-db in /opt/traccar/conf/traccar.xml:ro (/run/media/ippo/TOSHIBA/traccar/conf/traccar.xml in the example)

open a root mysql shell

`docker exec -it mariadb mysql -u root -p`

type your root password 

Hit enter

`CREATE DATABASE IF NOT EXISTS traccar-db`

hit enter

```
grant all privileges on `db_name`.* TO `user_name'@'%` identified by `root_pass`
```

hit enter

`flush privileges`

hit enter

```
ALTER DATABASE traccar CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
```

hit enter

\q

hit enter

`docker-compose up -d`


**periodic mysql backup and restore**

To backup  traccar with a date timestamp in the SQL file in order to have db "snapshots"

```
docker-compose -f  compose-file.yml exec dbname mysqldump -uroot -pYOUR_MARIADB_ROOT_PASSWORD --all-databases > dump-$(date +%F_%H-%M-%S).sql
```

And restore

```
docker-compose -f compose-file.yml exec -T dbname mysql -uroot -pYOUR_MARIADB_ROOT_PASSWORD  < mariadb-dump.sql
```


Put that on a crontab
And there you go you have periodic db snapshots


**Caddyfile**

*assuming you use caddy as a reverse proxy/ssl handler


`nano /etc/caddy/Caddyfile`

add 

```
#traccar
a.random.ddns.name.duckdns.org {
    reverse_proxy localhost:8082
}
```

**Create admin and disable new users**

First you habe to register as a normal user

From the login page
fill in email and prefered password
And register

Then you can log in with those credestria's

Its a good.idea for security reasons to disable sign ups

Login as default admin
User:pass = admin:admin

On top left corner 
Settings
Users
Select your user
Select the pencil icon
From the permissions tab enable admin


Then
From settings 
Server
On permissions tab disable registration


Log out and re-login with your created user 

From settings
Users
You can remove default admin now


**Traccar notifications**

I'm setting up notifications over telegram.
There are many notification types
Some interesting ones are enter / leave geofence (have to set up a geofence obvius'y first)
Device moving
Device stoped
Device online
Device offline

Useful if you monitor a second device e.g. a childs device

Edit the traccar.xml 

In he docker-compose example above I maped /run/media/ippo/YOSHIBA/conf/traccar.xml as the config location 

So 

`nano /run/media/ippo/TOSHIBA/conf/traccar.xml`

And add

```
<entry key='notificator.types'>web,mail,telegram</entry>
<entry key='notificator.telegram.key'>BOT_KEY</entry>
<entry key='notificator.telegram.chatId'>CHAT_ID</entry>
```

You have to create a bot
get its api token
Start a chat with it
And get the chat id


To create the bot
Visit @botfather
/newbot
Name it
And get the token


To get the chat id 
Type @get_id_bot in the conversation chat with the bot


Restart the container

You can test the connection visiting your acount and sending a test message

**public demo servers**

If you just want to test traccar before start building and selfhosting it, there are some public servers available 

Public servers (demo)
https://www.traccar.org/demo-server/

Android client app 
https://www.traccar.org/client/


Navigate to one of demo servers

demo4.traccar.org

demo3.traccar.org

demo2.traccar.org

demo.traccar.org

And register
Select username
Mail
And password


Then use registered mail and password to login


On client app
While service deactivated
Copy identifier
Insert the demo server URL you registered
Select location accuracy (high)
Select frequency report ~120 sec
Let others as is
and enable service

On status verify that the location is updating


On server
Top left menu
Select the plus icon to add a device
Select a random name
And the identifier from the client app
And save

The you can select the device on the top left menu

It appears on the map
From the refresh icon you can select a time period to show a location trail on the map and even animate the motion of tracks

From the reports tab you can you can view a table of location reports from a selected period

#disadvantages:
The public servers are ephemeral and your data will disapear at some point
you have to either buy a plan or selfhost the server

**Migrate google location history takeout to traccar**

[download your google maps location history takeout](https://takeout.google.com/takeout/custom/local_actions,location_history,maps,mymaps?)

- select location history only
- and json as format
- extract the Records.json

you'll use a python script to limit the resaults to just time , latitude and longitude converted to proper coordinates

```sh
git clone https://github.com/Scarygami/location-history-json-converter
cd location-history-json-converter
pip install -r requirements.txt
python location_history_json_converter.py Records.json  output.csv -f csv
```

The csv file generated by the script above will contain 3 columns:
- `time`
- `latitute`
- `longitude`

We'll add more columns:
- `deviceid`
- `protocol`
- `valid`

using the following command:
```sh
sed 's/^/osmand,1,/; s/$/,1/' export.csv > curated.csv
```

The csv file should now contain 6 columns:
- `@osmand`
- `@deviceid`
- `@time`
- `@latitude`
- `@longitude`
- `@valid`

These represent `tc_positions` table fields in traccar postgres db, so now the file should look like this:

```csv
protocol, 1, Time, Latitude, Longitude, 1
osmand, 1, 2012-08-25 21:26:20, 37.95954620, 23.72793730, 1
```

We are going to parse those values to as sql `LOAD DATA LOCAL INFILE` statement to the appropriate tc_position table fields

- copy the csv to the container
    ```sh
    docker cp curated.csv traccar-db:/
    ```

- open a root mysql shell to the db container
    ```sh
    docker exec -it traccar-db mysql -uroot -pROOTPASSWORD
    ```

- connect to database
    ```sh
    use traccar-db
    ```

- load the data:
    ```sql
    LOAD DATA LOCAL INFILE 'inn.csv' INTO TABLE tc_positions FIELDS TERMINATED BY ',' (@osmand, @deviceid,@Time,@Latitude,@Longitude,@valid) set protocol=@osmand,deviceid=@deviceid, devicetime=@Time,fixtime=@Time,servertime=@Time,latitude=@Latitude,longitude=@Longitude, valid=@valid;
    ```
