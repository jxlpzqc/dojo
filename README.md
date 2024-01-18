# DOJO

Deploy a pwn.hust.college dojo instance while pwn.hust.college is forked from pwn.college!

## Details

The pwn.hust.college dojo infrastructure is based on [CTFd](https://github.com/CTFd/CTFd).
CTFd provides for a concept of users, challenges, and users solving those challenges by submitting flags.
From there, this repository provides infrastructure which expands upon these capabilities.

The pwn.hust.college infrastructure allows users the ability to "start" challenges, which spins up a private docker container for that user.
This docker container will have the associated challenge binary injected into the container as root-suid, as well as the flag to be submitted as readable only by the the root user.
Users may enter this container via `ssh`, by supplying a public ssh key in their profile settings, or via vscode in the browser ([code-server](https://github.com/cdr/code-server)).
The associated challenge binary may be either global, which means all users will get the same binary, or instanced, which means that different users will receive different variants of the same challenge.

## Setup

```sh
curl -fsSL https://get.docker.com | /bin/sh
DOJO_PATH="./dojo"
git clone https://github.com/HUSTSeclab/dojo.git "$DOJO_PATH"
docker buildx install
docker build --load -t pwncollege/dojo "$DOJO_PATH"
docker run --privileged -d -v "${DOJO_PATH}:/opt/pwn.college:shared" -p 22222:22 -p 80:80 -p 443:443 --name dojo pwncollege/dojo
```

This will run the initial setup, including building the challenge docker image. It would build docker image based on the host architecture.
You can deploy dojo with [setup.sh](https://github.com/HUSTSeclab/dojo/blob/hustsec_dev/setup.sh).

> [!NOTE]
> Using buildx with Docker requires Docker engine 19.03 or newer.
> You can run `docker buildx install` to set buildx as the default builder.
> From [Docker driver specification](https://docs.docker.com/engine/reference/commandline/buildx_create/#driver), with the default docker driver,
> the `--load` flag is implied by default on `buildx build`.
> However, this is not true on the arm64 platform. So we add this option implicitly.

> [!NOTE]
> This command would map ports (22, 80, 443) in the container to the corresponding ports (22222, 80, 443) on the Docker host.
> If these ports are bound in you environment, you can disable these processes or revise these mapping ports.
> Please do not revise the ports 80 and 443, this will trigger an issue in ngnix-proxy.

### Local Setup

By default, the dojo will initialize itself to listen on and serve from `localhost.pwn.college` (which resolves 127.0.0.1).
This is fine for development, but to serve your dojo to the world, you will need to update this (see Production Setup).

It will take some time to initialize everything and build the challenge docker image.
You can check on your container (and the progress of the initial build) with:

```sh
docker exec dojo dojo logs
```

Once things are setup, you should be able to access the dojo and login with username `admin` and password `admin`.
You can change these admin credentials in the admin panel.

### Production Setup

Customizing the setup process is done through `-e KEY=value` arguments to the `docker run` command.
You can stop the already running dojo instance with `docker stop dojo`, and then re-run the `docker run` command with the appropriately modified flags.

In order to change where the host is serving from, you can modify `DOJO_HOST`; for example: `-e DOJO_HOST=localhost.pwn.college`.
In order for this to work correctly, you must correctly point the domain at the server's IP via DNS.

By default, a minimal challenge image is built.
If you want more of the features you are used to, you can modify `DOJO_CHALLENGE`; for example: `-e DOJO_CHALLENGE=challenge-mini`.
The following options are available:
- `challenge-nano`: A very minified setup.
- `challenge-micro`: Adds VSCode.
- `challenge-mini`: Adds a minified desktop (by default).
- `challenge-full`: The full (70+ GB) setup.

Because Dojo does not support external access via IP, we need to use the host's nginx for port forwarding (which also conveniently allows for domain configuration).

```sh
cp ./$IP.conf /etc/nginx/sites-enabled/
nginx -s reload
```

If you prefer not to use the `setup.sh` script for setup, here are the nginx forwarding rules I use as a reference. 
You just need to replace `$YOUR_HTTP_PORT` with the port you want to forward, `$IP` with your current IP, and `$MAP_HTTP_PORT` with the Docker-mapped port.

```conf
server {
    listen $YOUR_HTTP_PORT;
    listen [::]:$YOUR_HTTP_PORT;

    #listen $HTTPS_PORT ssl;
    #listen [::]:$HTTPS_PORT ssl;
    #ssl_certificate $CERT_FILE;
    #ssl_certificate_key $CERT_KEY;

    #server_name $DOMAIN;
    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $IP:$MAP_HTTP_PORT;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Accept-Encoding gzip;
        proxy_set_header Origin "http://$IP:$MAP_HTTP_PORT";
        proxy_buffering off;

        proxy_pass http://127.0.0.1:$MAP_HTTP_PORT;
    }
}
```

> [!NOTE]
> Regarding the nginx-proxy bug issue:\
> According to [this](https://github.com/nginx-proxy/nginx-proxy/discussions/2271#discussioncomment-8156338), 
> we can see that the latest version of nginx-proxy has changed the forwarding rule from $http_host to $host, 
> resulting in the inability to forward the port.\
> Therefore, we choose to roll back the nginx-proxy version to `1.3.1`.





## Customization

_All_ dojo data will be stored in the `./data` directory.

Once logged in, you can add a dojo by visiting `/dojos/create`. Dojos are contained within git repositories. 
Refer to [the example dojo](https://github.com/pwncollege/example-dojo) for more information.

## Contributing

We love Pull Requests! 🌟
Have a small update?
Send a PR so everyone can benefit.
For more substantial changes, open an issue to ensure we're on the same page.
Together, we make this project better for all! 🚀
