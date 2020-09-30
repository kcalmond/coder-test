
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
$ docker run -dt -p 127.0.0.1:8080:8080 \
-v "$HOME/.config:/home/coder/.config" \
-v "$PWD/coderproject:/home/coder/project" \
-u "$(id -u):$(id -g)" codercom/code-server:latest
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

## Using image built from code-server repo
* Cloned cdr/code-server locally
  * git clone https://github.com/cdr/code-server.git
  * Repo includes this release-image Docker file: https://github.com/cdr/code-server/blob/v3.5.0/ci/release-image/Dockerfile
  * Plan: use this Dockerfile to build an image and test docker run using it + find a set of docker commandline options that work with it
* Build image from locally cloned repo, using ../ci/release-image/Dockerfile and ../ci/release-image/build.sh as a guide:
  * Needed to install nvm & yarn
  ```
  $ wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
  $ export NVM_DIR="$HOME/.nvm"
  $ [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
  $ [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
  $ nvm ls-remote  # <-- find most recent "LTS" version
  $ nvm install 12.18.4
  $ node -v # <-- check node installed ok & version

  $ curl -o- -L https://yarnpkg.com/install.sh | bash
  ```

  * First attempted at build.sh failed at Dockerfile step 9: `COPY release-packages/code-server*.deb /tmp/`
  * Reviewed CI automation scripts and determined I needed to pre-populate the release-packages before running build.sh. Used `release-github-assets.sh` to download asset files into ../code-server/release-packages/
  * ran build.sh successfully - created code-server-arm64:3.5.0 image
  ```
  coder@hackberry:~/.config/code-server> docker images -a
  REPOSITORY          TAG                  IMAGE ID            CREATED             SIZE
  code-server-arm64   3.5.0                355f113aed56        4 hours ago         790MB
  ```
  * After a few attempts at the correct docker command to launch the server I found this version worked:
  ```
  $ id
  uid=1001(coder) gid=100(users) groups=100(users),479(docker)

  $ docker run -dt -p 8080:8080 \
  -v "$HOME/.config:/home/coder/.config" \
  -v "$PWD:/home/coder/project" \
  -u "1001:100" code-server-arm64:3.5.0
  11462498d268554c895d8f8c621c30be8335e145a0b110072b5b862456a30fa3

  coder@hackberry:~/.config/code-server> docker ps
  CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                                      NAMES
  11462498d268        code-server-arm64:3.5.0              "/usr/bin/entrypoint…"   6 seconds ago       Up 5 seconds        0.0.0.0:8080->8080/tcp                     jovial_kirch

  coder@hackberry:~/.config/code-server> docker logs 11462498d268
  whoami: cannot find name for user ID 1001
  [2020-09-30T00:45:13.066Z] info  Using config file ~/.config/code-server/config.yaml
  [2020-09-30T00:45:13.765Z] info  Using user-data-dir ~/.local/share/code-server
  [2020-09-30T00:45:13.785Z] info  code-server 3.5.0 b509063e143bbf74b74ec295260c4fd5f6332f71
  [2020-09-30T00:45:13.799Z] info  HTTP server listening on http://0.0.0.0:8080
  [2020-09-30T00:45:13.800Z] info      - Using password from ~/.config/code-server/config.yaml
  [2020-09-30T00:45:13.800Z] info      - To disable use `--auth none`
  [2020-09-30T00:45:13.800Z] info    - Not serving HTTPS
  ```
  ![](https://github.com/kcalmond/coder-test/blob/master/code-server_login.jpg | width=150)
  ![Code - OSS](https://github.com/kcalmond/coder-test/blob/master/Welcome_%E2%80%94_coder_%E2%80%94_Code_-_OSS.jpg | width=300)







## Using image: linuxserver/coder-server
Their image and docker command guidance worked. Was able to open coder server home page using exposed url/port.

* ref: https://hub.docker.com/r/linuxserver/code-server
* Adaptation of the run-in-docker example from above page:
```
$ docker run -dt -p 8443:8443 \
-v "$HOME/coderapp/config:/config" \
-e PUID=1001 -e PGID=100 -e TZ=America/Los_Angeles \
linuxserver/code-server
0a64bae936f5f625cc99a8f680d55992c8f0e179592e4e3960e57236e0c56daa

9d266976c7b6        linuxserver/code-server              "/init"             8 seconds ago       Up 7 seconds        0.0.0.0:8443->8443/tcp                     silly_allen

```
Container start up log output :
```
coder@hackberry:~/coderapp/config> docker logs 0a64bae936f5
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

User uid:    1001
User gid:    100
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
[2020-09-29T04:40:36.023Z] info  Using config file ~/.config/code-server/config.yaml
[2020-09-29T04:40:36.742Z] info  Using user-data-dir ~/data
[2020-09-29T04:40:36.754Z] info  code-server 3.5.0 de41646fc402b968ca6d555fdf2da7de9554d28a
[2020-09-29T04:40:36.765Z] info  HTTP server listening on http://0.0.0.0:8443
[2020-09-29T04:40:36.766Z] info    - No authentication
[2020-09-29T04:40:36.766Z] info    - Not serving HTTPS
coder@hackberry:~>
```
