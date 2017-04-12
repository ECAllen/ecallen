+++
title = "deploying blog"
draft = false
date = "2017-01-29T23:43:13-05:00"

+++

### Deployment

First create the jar files for the client and server

edit bright-paper-werewolves/src/clj/bright-paper-werewolves/server.clj

change the port to 4000

```shell

lein do clean, uberjar

```

create a uberjar for the blog server

change the port to 4010

```shell

lein do clean, uberjar

```

Deploy the jars to a web server. At this point there are many ways to get the server running (nginx, tomcat etc...) so I wont go into particulars. You also might want to look into heroku as an option.

I am going to create a systemd service file for each app then configure nginx

Creating the systemd service (unit) files

``` shell

[Unit]
Description=Blog server
After=syslog.target
After=network.target

[Service]
Type=simple
User=blog
ExecStart=

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```
