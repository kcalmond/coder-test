
## linuxserver.io

Successful build test inside docker using example/image from linuxserver.io.

* ref: https://hub.docker.com/r/linuxserver/code-server
* Adaptation of the "docker" usage provided above:
```docker run -dt -p 8443:8443 -v "$HOME/.config:/home/coder/.config" -e PUID=1000 -e PGID=1000 -e TZ=America/Los_Angeles linuxserver/code-server```

## coder code-server

* ref: https://github.com/cdr/code-server/blob/v3.5.0/doc/install.md
* Tried many forms of their docker start example:
```# This will start a code-server container and expose it at http://127.0.0.1:8080.
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

Could not connect to http service at exposed addr:port. Tried a few variations. Nothing worked. Got this output consistently when looking at docker log:

```coder@hackberry:~/project> docker run -dt -p 127.0.0.1:8080:8080 -v "$HOME/.config:/home/coder/.config" -v "$PWD:/home/coder/project" -u "$(id -u):$(id -g)" codercom/code-server:latest
7c7f39c61368ecb5f6a66ad9e61f2a57ec5760b50fc96753c3145cf301c58e27
coder@hackberry:~/project> docker ps
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                                      NAMES
7c7f39c61368        codercom/code-server:latest          "/usr/bin/entrypoint…"   7 seconds ago       Up 5 seconds        127.0.0.1:8080->8080/tcp                   adoring_kilby
4a9341c95853        rancher/rancher:v2.4.6-linux-arm64   "entrypoint.sh"          3 weeks ago         Up About an hour    0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   fervent_poincare
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
coder@hackberry:~/project> curl http://127.0.0.1:8080
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
coder@hackberry:~/project> docker exec -it 7c7f39c61368 /bin/bash
coder@7c7f39c61368:~$ curl http://127.0.0.1:8080
coder@7c7f39c61368:~$ ```

Found that coder image writes `` into config.yaml every time, even with this form of the docker run:

```
coder@hackberry:~/.config/code-server> rm config.yaml
coder@hackberry:~/.config/code-server> docker run -dt -p 8443:8443 -v "$HOME/.config:/home/coder/.config" -v "$PWD:/home/coder/project" -u "$(id -u):$(id -g)" codercom/code-server:latest
90392d5eb1b26c53c1c4d16210168b6601fab233f717862c1ce3850ed7822498
coder@hackberry:~/.config/code-server> ll
total 4
-rw-r--r-- 1 coder users 88 Sep 28 19:26 config.yaml
coder@hackberry:~/.config/code-server> cat config.yaml
bind-addr: 127.0.0.1:8080
auth: password
password: b4fadc0238fa5ac5adc899c1
cert: false
coder@hackberry:~/.config/code-server>
```
