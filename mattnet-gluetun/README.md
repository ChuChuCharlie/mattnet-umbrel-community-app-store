## Gluetun for UmbrelOS

This is an unofficial kludge to bring Gluetun support to allow Umbrel containers to work over VPN tunnels.

This is a best endevour project, and works for me but YMMV. Note this breaks the container DNS, so other apps may need updating. See the example below.

I'd love to include a simple web form or something that allows the user to enter their variables based on the VPN type/provider selected, but I'm not smart enough to come up with something like that so you have to do it by editing the configs yourself.

## How to use:

Install through the community app store.

Once installed, open the file 'exports.sh'. You need to set the variables according to your VPN provider, referring to the [Gluetun wiki](https://github.com/qdm12/gluetun-wiki) for specifics.

Edit the docker-compose.yml file. Check that APP_PORT doesn't overlap with anything else running on your Umbrel instance.

If you want to access your tunnelled app after it's been VPN'ed, under ports: add port redirects as required
E.g. if your app needs inbound TCP/UDP 8080, TCP 80 and TCP 443, you'll need 3 variables in exports.sh:
```
    export GLUETUN_TCP_PORT_1="3000"
    export GLUETUN_UDP_PORT_1="3000"
    export APP_TCP_PORT_1="8080"
    export APP_UDP_PORT_1="8080"
    export GLUETUN_TCP_PORT_2="3001"
    export APP_TCP_PORT_2="80"
    export GLUETUN_TCP_PORT_3="3002"
    export APP_TCP_PORT_3="443"
```
And then in docker-compose.yml:
```
    ports:
      - ${GLUETUN_TCP_PORT_1}:${APP_TCP_PORT_1}/tcp
      - ${GLUETUN_UDP_PORT_1}:${APP_UDP_PORT_1}/udp
      - ${GLUETUN_TCP_PORT_2}:${APP_TCP_PORT_2}/tcp
      - ${GLUETUN_TCP_PORT_3}:${APP_TCP_PORT_3}/tcp
```
Restart the Gluetun app by right clicking it (or, open the terminal and run 'umbreld client apps.restart.mutate --appId mattnet-gluetun')

You now need to tell the app yoou want to run through VPN to use the Gluetun container. This means editing the docker-compose.yml FOR THAT APP and adding this line
```
    server:
      ...
      network_mode: container:gluetun_server_1
```

EXAMPLE SETUP FOR QBITTORRENT

This is my config for how I set up qBittorrent, which seems to be working

exports.sh

```
export GLUETUN_TCP_PORT_1="3000"
export GLUETUN_UDP_PORT_1="3000"
export APP_TCP_PORT_1="8080"
export APP_UDP_PORT_1="8080"

export VPN_SERVICE_PROVIDER="airvpn"
export VPN_TYPE="wireguard"
#export VPN_INTERFACE="VPN interface name e.g. tun0"
#export OPENVPN_USER="your openvpn username"
#export OPENVPN_PASSWORD="your openvpn password"
#export OPENVPN_ENDPOINT_IP="your openvpn endpoint ip"
#export OPENVPN_ENDPOINT_PORT="your openvpn endpoint port"
export WIREGUARD_PRIVATE_KEY="abcd1234privatekey"
export WIREGUARD_PRESHARED_KEY="abcd1234presharedkey"
export WIREGUARD_ADDRESSES="1.2.3.4/32"
#export WIREGUARD_PUBLIC_KEY="your wireguard public key"
#export WIREGUARD_ENDPOINT_IP="your wireguard endpoint ip"
#export WIREGUARD_ENDPOINT_PORT="your wireguard endpoint port"
#export WIREGUARD_ALLOWED_IPS="your wireguard allowed ips"
#export WIREGUARD_IMPLEMENTATION="your wireguard implementation e.g. userspace or kernel"
export WIREGUARD_MTU="1320"
#export WIREGUARD_PERSISTENT_KEEPALIVE_INTERVAL="your wireguard persistent keepalive interval"
export SERVER_COUNTRIES="Austria, Belgium"
#export SERVER_REGIONS="your server regions"
#export SERVER_CITIES="your server cities"
#export SERVER_NAMES="your server names"
#export SERVER_HOSTNAMES="your server hostnames"
```

mattnet-gluetun/docker-compose.yml
```
version: '3.7'
services:
  app_proxy:
    environment:
      APP_HOST: gluetun_server_1
      APP_PORT: 3000
    container_name: gluetun_app_proxy_1
  server:
    image: qmcgaw/gluetun
    container_name: gluetun_server_1
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - ${GLUETUN_TCP_PORT_1}:${APP_TCP_PORT_1}/tcp
      - ${GLUETUN_UDP_PORT_1}:${APP_UDP_PORT_1}/udp
    environment:
      - PUID=1000
      - PGID=1000
      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER}
      - VPN_TYPE=${VPN_TYPE}
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_PRESHARED_KEY=${WIREGUARD_PRESHARED_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}
      - WIREGUARD_MTU=${WIREGUARD_MTU}
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
```

qBittorrent/docker-compose.yml
```
version: '3.7'
services:
  app_proxy:
    environment:
      APP_HOST: qbittorrent_server_1
      APP_PORT: 8080
      PROXY_AUTH_ADD: 'true'
    container_name: qbittorrent_app_proxy_1
  server:
    image: >-
      ghcr.io/hotio/qbittorrent:release-5.1.2@sha256:4c731e88dd419a20f0e158cef9672e902a7e2c71acea48b8989567f00b5fb095
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - ${APP_DATA_DIR}/data/config:/config
      - ${UMBREL_ROOT}/home/Downloads:/app/qBittorrent/downloads
    restart: on-failure
    network_mode: container:gluetun_server_1
    container_name: qbittorrent_server_1
```

You can't access qBittorrent using the app icon, but you can change the port to 3000

e.g. http://umbrel.local:3000

You can verify it's working by checking the logs (View->Log, and then on the right side clicking 'Execution log'. The external IP should be detected and not be your IP).

If any other apps use your VPN'ed app they need telling about the change. Example, Radarr/Sonarr under Settings->Download Clients, select qBittorrent and change the host and port.