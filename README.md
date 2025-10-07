# README

A simple seedbox setup using Gluetun, running on Podman.
For a friend.

## Prerequisites

Before installing this project, ensure you have the following:

- **Operating System**: Ubuntu 24.04
- **Podman**: Must be installed and configured
- **Docker Compose**: As the compose provider for Podman
- **Ruby**: >= v3.4.0 Required for running Rake tasks

## Installation & Setup

### Run Configure

```
./configure
```

This will install most of the packages needed.

### Copy in Your OpenVPN Files

Copy your OpenVPN configuration file into `./custom.devl.ovpn`. 
This will be treated as local Development OpenVPN file. 

Create another separate OpenVPN for your actual Seedbox, copy this to
`./custom.prod.ovpn`. 

We treat the actual seedbox as the Production environment.

NOTE: If your ovpn file contains a 'remote <address>' line
with a domain name instead of an IP address -- you must convert
this to an IP Address.

### Create a config.yaml

Copy `./config.sample.yaml` to `./config.yaml`,
and fill in the following fields:
- `remote_host` -- This should be SSH Host (alias) used that you use to SSH
  into your Seedbox.
- `prod_hostname` -- This should be the actual Hostname of your Seedbox. This
  must match the output of running `hostname` on the Seedbox.
- `env.BT_DOWNLOADS_PATH` -- This should be the local path to your Torrent
  download folder. This will be mounted into the torrent service container.


## Usage (Local/Development)

First, to ensure that everything is working. Start the system in Development on your local machine.
To do so run:

```bash
rake start
```

Once everything is up. Run `rake ss_start` -- This will create start a ss-local proxy to connect the seedbox.

Create a new Firefox profile, with SOCKS-5 Proxy set to localhost port 1080.

Once Firefox is connected through your SOCKS-5 Proxy,
proceed to leak testing your outbound ip with a site like [IPLeak.net](https://ipleak.net/).

## Usage (Seedbox/Production)

To start using this on your Seedbox, you must first upload it. 
To do this run:

```bash
rake upload
```

This will upload this directory to the same directory on your seedbox relative to $HOME.

To ensure that everything works just the same on your seedbox, run `rake start` once again
and ensure that everything comes up.
Once you are confident things are working, Hit Ctrl+C to shut it down.

### Service Install

Now we can proceed to install this as a systemd service. Run:

```bash
rake service_install
```

This will install a systemd service named `sbox1.<username>.service` which
effectively runs `rake start`. It enable the service to start on boot. And it
will start the service immediately.

How do I access QBitorrent on my remote Seedbox? 
How do I browse the web through the VPN that is running on the Seedbox?

To make connect to the services running on your seedbox, run `rake connect`. 
This will create an SSH tunnel with forwarded ports for the QBittorrent WebUI and Shadowsocks. 

You should be able to reach the QBittorrent WebUI at http://localhost:9092.
If you now launch your SOCKS-5 proxy configured Firefox, which is expecting a SOCKS-5 Server
at localhost:1080, it will also work. 

### Service Uninstall 

To remove/uninstall the systemd service, simply run:

```bash
rake service_uninstall
```

## Services + Ports Open

- [QBitorrent WebUI on Port 9092](http://localhost:9092/?)
- ShadowSocks on Port 8388
- HTTP Proxy on Port  8888


