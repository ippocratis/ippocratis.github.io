---
layout: post
title:  "Libre photos"
author: ippo
categories: [ server, photos]
image: assets/images/banner.png
---


A self-hosted open source photo management service that serves ad a hacienda for the android client uhuruphotos.

**so what?**

Well uhuruphotos is what was missing from all selfhosting photo galleries

A fast and  beatufull google photos alternative

It caches photos so there is offline view for cached items

Fast image loads means a well writen app and a.well writen librephotos api

Fast scrolling and thumbnail loads on time feed

[Project page](https://github.com/savvasdalkitsis/uhuruphotos-android)

**backround info**

Its been for a long time that google had provided an excelent service for free.

Unlimited cloud storage for photos at a fair analysis / size (16mp).

Its been a while that they have changed their charging policy and limited the storage to just 15gb at user have to pay above that li.it.

So the need for an android app that can sync with self hosted gallerries and be as  good as google photos in displaying photos timeline 

and providing helpfull  features like photos on map and face tags arise.

There where already good selfhosted softwhare around , great projects like photoprism , photoview AND librephotos but with a major flow.

There was no android app.

Well to be more precise photoprism and librephotos have android apps but are proof of concept project incomplete and unusable.

That means you only have a webui.

Progressive web apps are OK for certain jobs but not for images galleries.

Photos are not cached and lag is furstrating as everything gets reloaded all the time



**What we'll walk through**

- How to get librephotos server running on your raspberry
- Basic requirements for securelly remotely access your librephotos server
- Auto upload photos from phone to raspberry
- Auto categorise photos in your gallery folder by year,month,day
- Auto scan and import photos from your gallery folder with librephotos
- Setup Uhuru photos

**requirements**

- Open ports on your router 80 and 443 and port forword these ports in to your raspberry Lan ip

That is specific to your router model and requires that your router is not behind a Nat by your isp

- A free dynamic subdomain from a provider
e.g. duckdns

Navigate to http://www.duckdns.org/ 

For quick signup use oauth from the provider list e.g. github

On domains use a subdomain name e.g. "libre" and click add domain . now your domain is https://libre.duckdns.org 

and it will link to your raspberry

To auto audate it use a cronjob

`*/5 * * * * sleep 38 ; curl -s "https://www.duckdns.org/update?domains=libre.duckdns.org&token=your-duckdns-token"`

Your duckdns token is on your duckdns acount page

- A web server to reverse proxy requests from the dynamic subdomain IP to the localhost port librephotos is running and handling SSL certs

For installing caddy web server and how it handles SSL read.my blogpost [here](http://depl0yed.herokuapp.com/blog/caddy2)

In our example all you need to do is edit the Caddyfile

`nano /etc/caddy/Caddyfile`

add

`libre.duckdns.org {
    reverse_proxy localhost:3000}`

To it

And restart caddy

`sudo systemctl restart caddy`

Assuming your domain is libre.duckdns.org and librephotos is running in 3000 port

- Docker and docker-compose plus python

To install docker on raspberry

`>curl -fsSL https://get.docker.com -o get-docker.sh`

Then

`>sudo sh get-docker.sh`

To install docker-compose

`sudo pip3 -v install docker-compose`

[Guide](https://dev.to/rohansawant/installing-docker-and-docker-compose-on-the-raspberry-pi-in-5-simple-steps-3mgl)

**get librephotos server running on your raspberry**

[Link](https://docs.librephotos.com/1/)

Clone the repo

`git clone https://github.com/LibrePhotos/librephotos-docker.git`

CD to the folder

`cd librephotos-docker`

Copy .env template

`cp librephotos.env .env`

Edit it

`nano .env`


```
# Location of your photos.

myPhotos=/run/media/user/TOSHIBA/photos/
```

The photos folder filepath
e.g.
assuming you store your photos on an external Toshiba drive

```
# Comma delimited list of patterns to ignore

skipPatterns=.json
```
If you have file types you want to skip scan

```                                                     
# The location where you would like to store the log files. The full path must exist as it will not be created for you.

logLocation=/run/media/user/TOSHIBA/uhuru-logs          
```
Again I use my external drive as the logs location to be stored

```
# Protected media directory. This is where we store some files like the thumbnails of your images.

proMedia=/run/media/ippo/TOSHIBA/media
```

Thumbnails folder again on external drive

```
# Where shall we store the cache files.

cachedir=/run/media/ippo/TOSHIBA/cache
```
Cache files , same                                      

 ```
dbLocation=/run/media/ippo/TOSHIBA/uhuru-db
```

Database filepath , same

```                                                     
# Username for the Administrator login.

username=user
````

Your username

```                                                                                                  
# Password for the administrative user you set above.

userPass=pass
```
Your password

```
# Secret Key. Get one here https://rb.gy/emgmwo (Shortened random.org link)             

# Secret Key. Get one here https://rb.gy/emgmwo (Shortened random.org link)

shhhhKey=globefish-goes-here
```

Generate a key from the link and paste here .used to encrypt the db

```                                                     
#What port should Libre Photos be accessed at (Default 3000)

httpPort=3333
```
Just use an available port on your raspberry
You can scan for used ports with nmap

`sudo nmap -sT -p- 192.168.1.x`
Where 192.168.1.x your your raspberry lan ip
Use a not used port

```                                                                                                   

# Get a Map box API Key https://account.mapbox.com/auth/signup/   
# Get a Map box API Key https://account.mapbox.com/auth/signup/

mapApiKey=pk.yourmapboxkey
```

sign up at mapbox and get an api . mapbox is optional and alternative to google maps

```                                                     
# What branch should we install the latest weekly build or the development branch (dev)

tag=dev
```

Choose your branch latest (weekly) or dev (rolling)

```                                                                                                      
# You can set the database name. Did you know Libre Photos was forked from OwnPhotos?

dbName=librephotos
```
Your dB name

```
# Here you can change the user name for the database.

dbUser=docker
```

DB username

```                                                                                                   
# The password used by the database.                              

dbPass=pass
```

DB password

```                                                                                                     
# Number of workers, which take care of the request to the api. This setting can dramatically affect the ram usage.     
# A positive integer generally in the 2-4 x $(NUM_CORES) range.
# You’ll want to vary this a bit to find the best for your particular workload.
# Each worker needs 800MB of RAM. Change at your own will. Default is 2.

gunniWorkers=2
```
Number of workers that handles api requests (talk with the android client
                                                        
```
# Number of workers, when scanning pictures. This setting can dramatically affect the ram usage.
# Each worker needs 800MB of RAM. Change at your own will. Default is 1.

HEAVYWEIGHT_PROCESS=2
```
Number of workers that handles the photo scans
I found fit for my RPI400 to use 2 workers for both cases

                                                        
**Start containers**

The docker-compose.yml is filed with our env vars from the .env file
you can review it

`cat docker-compose.yml`

Run the containers as daemons

`docker-compose up -d`

App is available at localhost:3333  or otherwise specified port (default 3000)

> note that librephotos api needs the folder contaning the ohotos to have 7 permissions for others

`sudo chmod 777 -R ~/photos` is what i did

**Update images**

CD to librephotos-docker folder and Stop containers

`docker-compose down`

Pull the new images

`docker-compose pull`


And re-compose

`docker-compose up -d`

**some remarks**

For face classification you'll have to access the webui tag some pics and name some faces and then train it

then you can verify/add from the infered tab

Auto import from phone to an import folder and then auto organise them to folders by year month day and move them to at image library .
Then scan our library with librephotos

**periodic postgres backups (snapshots) and restore**
                                                              
create a folder to store the backups           
                                                                            
 `mkdir db-backup`            
                                 
cd to the folder
                                                              
`cd db-backup`
                                                              
Grab postgres container id

`docker ps`
                                                              
e.g.40b12de38945                
                                                                                                            
**Backup**
                                                                      
the command should look like:           
                                                                             
`docker exec -t your-db-container pg_dumpall -c -U your-db-user > dump_$(date +%Y-%m-%d_%H_%M_%S).sql`            
         
your-db-container is the postgres contaimer id     
                                                                                         
your-db-user is docker in .env

unless it was changed             
                                                                                                                 
A file like dump_20-11-2022_00_55_26.sql is created   

**Restore**               
                                                                                                                      
`cat your_dump.sql | docker exec -i your-db-container psql -U your-db-user -d your-db-name`           
                                                                            
again                                                 

your-db-container is the postgres contaimer id                                                                                             

your-db-user is docker in .env unless it was changed
                                                                      
your-db-name is librephotos in the .env unless it was changed

so in our example the command should be

`cat dump_20-11-2022_00_55_26.sql | docker exec -i 40b12de38945 psql -U docker -d librephotos`

add the backup command 
                                                                    
to a crontab and take db snapshots @daily , @weekly etc        

e.g.

`crontab -e`

`@weekly docker exec -t 40b12de38945 pg_dumpall -c -U docker > /path/to/where/you/save/your/backups/dump_$(date +%Y-%m-%d_%H_%M_%S).sql`

another idea is to scan periodicaly the db backup folder and delete very old backups to keep size at a minimum

you can do this with find , ntime and rm

e.g.

`find /path/to/where/you/save/your/backups -type f -mtime +30 -exec rm -f {} \;`

this will delete entries older than 30 days in that folder

**auto import from phone**

> note that the android app aims to add auto upload but atm it is not available.
the bellow procedure will be unnecesery when auto upload becomes available

needs a webdav server e.g. webdav nononsense docker image. Read my "webdav" post about running a webdav server on raspberry 

[here](http://depl0yed.herokuapp.com/blog/webdav+)

create a folder named photoimport and chmod 777 it

For syncing I use foldersync android app it is a paid app but its the best for the job and you can sync with the full featured 

free version just fine if you can stand some adds

Also the Dev can provide app licence for 3 devices outside the playstore for devices without google services

worth the money

[Link](https://www.tacit.dk/foldersync)

**app settings**

On foldersync android app add account
select webdav
add your webdav ddns https url , username and password

On folderpairs add a pair

In general tab

On local select your dcim folder

On remote select photoimport folder

On sync type sect "to remote"

On scheduling tab
enable scheduled sync daily

On sync options tab
enable sync subfolders

On advanced tab
enable retry sync
and disable temp file scheme
Also on "one way sync options" enable "only resync source if modified (ignore target deletion)
As we are gonna move the files from target folder (photoimport folder) to our photo gallery after we organise them with exit tool.

On conection tab
as desired but i guess it makes.sense to sync only.on wifi

On notification
enble all


**auto organise with exif tool**

download and install exiftool for unix from

`https://exiftool.org/index.html`

`cd`

`gzip -dc Image-ExifTool-12.42.tar.gz | tar -xf -`

`cd Image-ExifTool-12.42`

`perl Makefile.PL`

`make test`

 `sudo make install`

The exiftool command should be like

`exiftool '-Directory<CreateDate' -d /destination/directory/%Y/%m   /source/directory`

we can create a script that runs this command

Later we will call this script in a crontab

directly executing exiftool in crontab fails

So

`nano exiftool-photos.sh`

and paste

```
#!/bin/bash
sudo /usr/bin/site_perl/exiftool '-Directory<CreateDate' -d /run/media/ippo/TOSHIBA/photos/%Y/%m  /run/media/ippo/TOSHIBA/photoimport;
```

Added a hashbag in the beginning of the script and calling the full path of the exiftool executable

Then create a cron job

`export VISUAL=nano; crontab -e`

And paste

`# Every day at 2 AM
0 2 * * * bash /home/ippo/photos-exif.sh`

Or the full path of where ever your script is

**Autoscan server for new photos**


The docker command is

`sudo docker exec --user root CONTAINER_NAME python3 manage.py scan`

CONTAINER_NAME is the librephotos backend
you can get it with

 `docker ps`

it should be

`brephotos-docker-backend-1`

in this case the command is

`sudo docker exec --user root brephotos-docker-backend-1 python3 manage.py scan`

We can create a cron job to regularly call this command

Create a new crontab with nano

`export VISUAL=nano; crontab -e`

And paste

`# Every day at 3 AM
0 3 * * * sudo docker exec --user root brephotos-docker-backend-1 python3 manage.py scan >/dev/null 2>&1`


**timeline**

- Our photos are transfered directly  from our phone to the photoimport folder  as long as we are connected to WiFi (or any network)

- organised to y/m/d and moved to image library every day at 2

- image library scaned from librephotos everyday at 3

**Android app**

Uhuruphotos
[GitHub project link](https://github.com/savvasdalkitsis/uhuruphotos-android)

As stated there

The app is currently in closed beta on the Google Play store.
You have to Contact the Dev  on twitter for access and playstore link.

[Twitter link](https://twitter.com/geeky_android)

*update:

for installs uotside the playstore apk releases are available at  [github](https://github.com/savvasdalkitsis/uhuruphotos-android/releases)

**About the app**

While still early days, it already has a lot of features:

- Photo feed with multiple views which can be changed by pinch to zoom gestures

- Multiple select in feed to share/delete multiple items at once

- Periodic background synchronization with LibrePhotos server


- Photo details view with information like date, location, gps map, people view, sharing, adding/removing 

from favourites (synced with LibrePhotos server)

- Video details view with all the above features and video playback

- Search your photos using LibrePhotos' search engine. Get search suggestions based on your photos.

- People view and suggestions for people with most photos.

- Photo map. See a heatmap of your photos. Navigate around the globe with the interactive map and see photos taken 

in the location currently viewed.


- User created and auto generated albums from LibrePhotos

- Dark/Light mode (manual and auto)

- Tablet support

- A lot of settings to help you control the app storage and memory requirements along with how frequently to perform synchronization 

with the LibrePhotos server.

- up to 2GB of images cache

-  up to 2GB of video cache

- Fingerprint authentication to lock the app


As mentioned above, a lot of features will soon be implemented before the full public release, mainly:


- Local photo support. This will make UhuruPhotos a viable Google Photos alternative which can be used as your primary camera roll viewer.

- Backup/Sync local images with librephotos server. Take control of your data by never having to worry about photo backups.

- Basic photo editing capabilities.

- Biometric authentication

all you need to access your photos is your dynamic dns subdomain through which you access your server
libre.duckdns.org in our example
the libre photos user name and password as provided in the .env file when composing it
