# docker-compose with separate VPN server
The following guide will provide instructions on running the backend, fronend,
and db from docker-compose along with separate servers for VPN and docker
targets as can be seen on the diagram below.

<center><img src="https://raw.githubusercontent.com/echoCTF/echoCTF.RED/master/docs/assets/docker-compose-including-vpn-topology.png" alt="echoCTF.RED docker-compose-novpn diagram" height="400px"/></center>

In this guide the infrastructure assumes a separate servers/vms. The servers described are as following:

* **Linux Host**: The linux host that the `docker-compose` command will run on. The file `docker-compose-novpn.yml` builds and starts containers for  `frontend`, `backend` and `db` on a Linux host as can be seen on the diagram below.

* **VPN Host**: An OpenBSD gateway host for VPN and docker servers and target
containers configured following the instructions from [VPN Server Installation](/docs/VPN-SERVER.md).

* **Docker Server**: A linux debian host (dockerd160 on the diagram above) that we will utilize as docker API server and that the actual target containers will run on (_target1 and target2 on the diagram above_).

The following networks are used throughout the document
* `echoctfred_public`: `172.26.0.0/24`
* `echoctfred_private`: `172.24.0.0/24`
* `echoctfred_targets`: `10.0.160.0/24`
* `OpenVPN`: `10.10.0.0/16`

Furthermore the following ports are mapped on the host server and containers
* tcp 0.0.0.0:8082 => echoctfred_backend 172.26.0.2:80
* tcp 0.0.0.0:8080 => echoctfred_frontend 172.26.0.3:80
* tcp 0.0.0.0:3306 => echoctfred_db 172.24.0.253:3306
* tcp 0.0.0.0:11211 => echoctfred_db 172.24.0.253:11211

The following volumes are configured and used
* `echoctfred_data-mysql` For persistent mysql data
* `echoctfred_data-challenges` under backend & frontend `/var/www/echoCTF.RED/*/web/uploads`
* `./themes/images` under `/var/www/echoCTF.RED/*/web/images` for logos and images

Before you start building you are advised to generate a Github OAuth Token to
be used by the composer utility.

This is only needed once and we do it in order to avoid hitting Github rate limits on their API, which is used by `composer`.
More information about generating a token to use can be found at [Creating a personal access token for the command line](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)

Once you've generated your token you can build the images.

_Keep in mind that this may require a lot of memory to run (our tests are performed on systems with at least 8GB ram)._

```sh
GITHUB_OAUTH_TOKEN=MY_TOKEN_HERE docker-compose -f docker-compose-novpn.yml build
```

From this point on the installation deviated depending on whether or not you want to have a dedicated ethernet interface for the `private` network, used for communication between `vpn` and `db`.

## Communication through docker exported ports
<center><img src="https://raw.githubusercontent.com/echoCTF/echoCTF.RED/master/docs/assets/docker-compose-including-vpn-explained-topology.png" alt="docker-compose including vpn with explanation"/></center>

Start the containers on the _Linux Host_ to bring up frontend, backend and db containers (as illustrated on the diagram below).
```sh
docker-compose -f docker-compose-novpn.yml up
# or start in detached mode
docker-compose -f docker-compose-novpn.yml up -d
```
<center><img src="https://raw.githubusercontent.com/echoCTF/echoCTF.RED/master/docs/assets/docker-compose-novpn-topology.png" alt="docker-compose-novpn-topology" height="400px"/></center>

Follow the instructions of [VPN-SERVER.md](/docs/VPN-SERVER.md) and adapt your values accordingly.

Follow the instructions from [DOCKER-SERVERS.md](/docs/DOCKER-SERVERS.md) to prepare your docker api server (dockerd160).

If you followed the instructions correctly your network topology should look like the diagram.


The _Linux Host_  `backend` will need to be able to access the Docker API Servers behind the `vpn` server. For this reason we will have to add a network route for the network `10.0.160.0/24` on the Linux Host by executing
```sh
route add -net 10.0.160.0/24 gw <vpn_public_ip>
```

## Dedicated ethernet interface for private network
There is also an alternative setup for providing a dedicated network interface for the private network `private` (color red). This involves the addition of an extra ethernet adapter on both vpn and linux host.

<center><img src="https://raw.githubusercontent.com/echoCTF/echoCTF.RED/master/docs/assets/docker-compose-including-vpn-dedicated-topology.png" alt="docker-compose-including-vpn-dedicated-topology" height="400px"/></center>

Ensure that all containers are stopped by running the following from the Linux Host
```sh
docker-compose -f docker-compose-novpn.yml down
```

Decide what is the ethernet interface you will dedicate for the `private` network (`enp2s0` in our example) and bring the containers up using the `docker-compose-novpn-macvlan.yml`
```sh
PRIVATE_PARENT_INTERFACE=enp2s0 docker-compose -f docker-compose-novpn-macvlan.yml up
# or
export PRIVATE_PARENT_INTERFACE=enp2s0
docker-compose -f docker-compose-novpn-macvlan.yml up
```

Add an additional ethernet adapter to the VPN host (em2 in our case). Once added configure the interface by running the following commands from the vpn server
```sh
echo "inet 172.24.0.1 255.255.255.0 NONE group private">/etc/hostname.em2
sh /etc/netstart em2
```

If you are using virtual machines make sure that you allow promiscues mode to the interface you dedicate for the macvlan bridge.
<center><img src="https://raw.githubusercontent.com/echoCTF/echoCTF.RED/master/docs/assets/vbox-network-settings.png" alt="Virtualbox Network Settings for macvlan and gateway" width="500px"/></center>

For VMware related options you can look at the following link [Configuring promiscuous mode on a virtual switch or portgroup (1004099)](https://kb.vmware.com/s/article/1004099)