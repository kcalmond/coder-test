
## Using image: codercom/code-server:latest

Coder docker command example from https://github.com/cdr/code-server/blob/v3.5.0/doc/install.md ...
```
# This will start a code-server container and expose it at http://127.0.0.1:8080.
# It will also mount your current directory into the container as `/home/coder/project`
# and forward your UID/GID so that all file system operations occur as your user outside
# the container.
#
# Your $HOME/.config is mounted at $HOME/.config within the container to ensure you can
# easily access/modify your code-server config in $HOME/.config/code-server/config.json
# outside the container.
mkdir -p ~/.config
docker run -it -p 127.0.0.1:8080:8080 \
  -v "$HOME/.config:/home/coder/.config" \
  -v "$PWD:/home/coder/project" \
  -u "$(id -u):$(id -g)" \
  codercom/code-server:latest
```

Tried this form of the command per above:
```
coder@hackberry:~/project> docker run -dt -p 127.0.0.1:8080:8080 -v "$HOME/.config:/home/coder/.config" -v "$PWD:/home/coder/project" -u "$(id -u):$(id -g)" codercom/code-server:latest
7c7f39c61368ecb5f6a66ad9e61f2a57ec5760b50fc96753c3145cf301c58e27

coder@hackberry:~/project> docker ps
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                                      NAMES
7c7f39c61368        codercom/code-server:latest          "/usr/bin/entrypoint…"   7 seconds ago       Up 5 seconds        127.0.0.1:8080->8080/tcp                   adoring_kilby

```
Cannot connect to http service at exposed addr:port. Tried a few variations of the above command w/o luck. Got this output consistently on startup of the coder-server image in container log:

```
coder@hackberry:~/project> docker logs 7c7f39c61368
whoami: cannot find name for user ID 1001
[2020-09-28T19:11:45.094Z] info  Wrote default config file to ~/.config/code-server/config.yaml
[2020-09-28T19:11:45.099Z] info  Using config file ~/.config/code-server/config.yaml
[2020-09-28T19:11:45.793Z] info  Using user-data-dir ~/.local/share/code-server
[2020-09-28T19:11:45.806Z] info  code-server 3.5.0 de41646fc402b968ca6d555fdf2da7de9554d28a
[2020-09-28T19:11:45.818Z] info  HTTP server listening on http://0.0.0.0:8080
[2020-09-28T19:11:45.819Z] info      - Using password from ~/.config/code-server/config.yaml
[2020-09-28T19:11:45.819Z] info      - To disable use `--auth none`
[2020-09-28T19:11:45.819Z] info    - Not serving HTTPS
```
No output from this curl command: `coder@hackberry:~/project> http://127.0.0.1:8080`
The http server appears to be responding, just nothing generated:
```
coder@hackberry:~/project> curl -v http://127.0.0.1:8080
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.66.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 302 Found
< Content-Type: text/plain
< Location: ./login?to=%2F
< Date: Mon, 28 Sep 2020 19:12:52 GMT
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 127.0.0.1 left intact
```
Tried exec'ing into the container to see if curl output is any different. No output that way either:
```
coder@hackberry:~/project> docker exec -it 7c7f39c61368 /bin/bash
  coder@7c7f39c61368:~$ curl http://127.0.0.1:8080
  coder@7c7f39c61368:~$
```


Found that coder image writes "bind-addr: 127.0.0.1:8080" into config.yaml every time, even with this form of the docker run (changed to `-p 8443:8443`):

```
coder@hackberry:~/.config/code-server> rm config.yaml

coder@hackberry:~/.config/code-server> docker run -dt -p 8443:8443 -v "$HOME/.config:/home/coder/.config" -v "$PWD:/home/coder/project" -u "$(id -u):$(id -g)" codercom/code-server:latest
90392d5eb1b26c53c1c4d16210168b6601fab233f717862c1ce3850ed7822498

coder@hackberry:~/.config/code-server> ls -l
total 4
-rw-r--r-- 1 coder users 88 Sep 28 19:26 config.yaml

coder@hackberry:~/.config/code-server> cat config.yaml
bind-addr: 127.0.0.1:8080
auth: password
password: b4fadc0238fa5ac5adc899c1
cert: false
coder@hackberry:~/.config/code-server>
```

## Using image: linuxserver/coder-server
Their image and docker command guidance worked. Was able to open coder server home page using exposed url/port.

* ref: https://hub.docker.com/r/linuxserver/code-server
* Adaptation of the run-in-docker example from above page:
```
coder@hackberry:~> docker run -dt -p 8443:8443 -v "$HOME/.config:/home/coder/.config" -e PUID=1000 -e PGID=1000 -e TZ=America/Los_Angeles linuxserver/code-server
9d266976c7b6cdac46ba703d737374b2fe4ed0d837825e5dd20e1881f9665a90
coder@hackberry:~> docker ps
CONTAINER ID        IMAGE                                COMMAND             CREATED             STATUS              PORTS                                      NAMES
9d266976c7b6        linuxserver/code-server              "/init"             8 seconds ago       Up 7 seconds        0.0.0.0:8443->8443/tcp                     silly_allen
4a9341c95853        rancher/rancher:v2.4.6-linux-arm64   "entrypoint.sh"     3 weeks ago         Up 9 hours          0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   fervent_poincare
```
Note difference in container start log output :
```
coder@hackberry:~> docker logs 9d266976c7b6
[s6-init] making user provided files available at /var/run/s6/etc...exited 0.
[s6-init] ensuring user provided files have correct perms...exited 0.
[fix-attrs.d] applying ownership & permissions fixes...
[fix-attrs.d] done.
[cont-init.d] executing container initialization scripts...
[cont-init.d] 01-envfile: executing...
[cont-init.d] 01-envfile: exited 0.
[cont-init.d] 10-adduser: executing...

-------------------------------------
          _         ()
         | |  ___   _    __
         | | / __| | |  /  \
         | | \__ \ | | | () |
         |_| |___/ |_|  \__/


Brought to you by linuxserver.io
-------------------------------------

To support LSIO projects visit:
https://www.linuxserver.io/donate/
-------------------------------------
GID/UID
-------------------------------------

User uid:    1000
User gid:    1000
-------------------------------------

[cont-init.d] 10-adduser: exited 0.
[cont-init.d] 30-config: executing...
[cont-init.d] 30-config: exited 0.
[cont-init.d] 99-custom-scripts: executing...
[custom-init] no custom files found exiting...
[cont-init.d] 99-custom-scripts: exited 0.
[cont-init.d] done.
[services.d] starting services
[services.d] done.
starting with no password
[2020-09-29T03:27:43.162Z] info  Wrote default config file to ~/.config/code-server/config.yaml
[2020-09-29T03:27:43.168Z] info  Using config file ~/.config/code-server/config.yaml
[2020-09-29T03:27:43.865Z] info  Using user-data-dir ~/data
[2020-09-29T03:27:43.877Z] info  code-server 3.5.0 de41646fc402b968ca6d555fdf2da7de9554d28a
[2020-09-29T03:27:43.889Z] info  HTTP server listening on http://0.0.0.0:8443
[2020-09-29T03:27:43.889Z] info    - No authentication
[2020-09-29T03:27:43.890Z] info    - Not serving HTTPS

coder@hackberry:~>
```
