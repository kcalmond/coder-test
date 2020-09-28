
Successful build test inside docker using example/image from linuxserver.io.

ref: https://hub.docker.com/r/linuxserver/code-server

Adaptation of the "docker" usage:

`docker run -dt -p 8443:8443 -v "$HOME/.config:/home/coder/.config" -e PUID=1000 -e PGID=1000 -e TZ=America/Los_Angeles linuxserver/code-server`

more changes...
