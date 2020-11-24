---
title: 'Free Small Matrix Server'
author: Paul Tötterman <paul.totterman@iki.fi>
license: CC-BY-NC-SA
---
# Free Small Matrix Server

Ingredients:

- Domain
- Server
- Software
- Configuration

## Free domain name

Get one from [Freenom] or [DuckDNS] for free or any other place you wish.
Freenom only gives free domains for a year. A Matrix server cannot be [migrated]
from one domain to another, so you might want to get a longer-lived one.

- https://tld-list.com/ - find cheap domains, look at renewal fees also
- https://www.namecheap.com/promos/amazing98s/ - cheap first year

You also need to specify subdomains (which is why most dynamic dns services
aren't sufficient). To do this I added the domain on the free [Cloudflare] plan.
You can also use freenom dns management if you prefer.

[Freenom]: https://www.freenom.com
[DuckDNS]: https://www.duckdns.org/
[Cloudflare]: https://www.cloudflare.com
[migrated]: https://github.com/matrix-org/synapse/issues/1209

## Get a free server

Comparison of free cloud offerings (2019/10):

| Vendor   | Time-limit | Count                    | RAM (GB) | Storage (GB) | Transfer (GB) |
| -------- | ---------- | ------------------------ | -------: | -----------: | ------------: |
| [AWS]    | 12 months  | 1 t2.micro               | 1        | 30           | 15            |
| [Azure]  | 12 months  | 1 B1S                    | 1        | 2x 64        | 15            |
| [GCP]    | no limit   | 1 f1-micro               | 0.6      | 30           | 1             |
| [Oracle] | no limit   | 2 VM.Standard.E2.1.Micro | 1        | 100          | 10000         |

[AWS]: https://aws.amazon.com/free/
[Azure]: https://azure.microsoft.com/en-us/free/
[GCP]: https://cloud.google.com/free/
[Oracle]: https://www.oracle.com/cloud/free/

If you choose to, register on Oracle Cloud (requires credit card for
verification) and create a free VM:

1. Create a VM instance
2. Give it a name
3. OS/image: Canonical Ubuntu 18.04 Minimal
4. Show Shape, Network and Storage Options -> Assign a public IP address
5. Add SSH key ([generate one][sshkey] using `ssh-keygen` if you don't have one)
6. Show advanced options. I removed monitoring, since I'll remove the monitoring
   agent later.

View resources and make sure the instance and boot volume are "Always Free" if
you don't intent to pay for them.

Make sure you can ssh to the server (using the ssh key you generated/picked)
using the user 'ubuntu' and can use `sudo`.

[sshkey]: https://docs.oracle.com/en/cloud/iaas/compute-iaas-cloud/stcsg/generating-ssh-key-pair.html

## Open firewall

### TURN/STUN and coturn

TURN/STUN is used for audio and video stream NAT/firewall traversal. coturn
implements the protocol. If you don't need 1-on-1 video/audio calls, or if
you're sure NAT and firewall won't be a problem (e.g. you only do calls on
networks where you're sure traffic can flow) you can disable coturn to save some
more RAM. Generally using coturn gives you much more reliable audio/video calls.

### Oracle Cloud Security lists

View resources -> Instances -> Select instance -> Virtual Cloud Network ->
Public Subnet -> Security Lists -> Default -> Ingress

Open incoming for CIDR 0.0.0.0/0:
- 22/tcp for SSH (should be open already)
- 80/tcp for HTTP
- 443/tcp for HTTPS
- 8448/tcp for Matrix federation
- 3478/tcp, 5349/tcp, 3478/udp, 5349/udp, 49152-49172/udp for TURN/STUN

### Ubuntu iptables

The Oracle Cloud Ubuntu images come with somewhat restrictive iptables rules by
default. Docker manages the instance firewall and we have the Oracle Cloud
firewall in front, so let's remove the current firewall to avoid trouble:

```sh
apt purge netfilter-persistent iptables-persistent
```

## Remove useless stuff (optional)

Oracle cloud includes a somewhat heavy monitoring daemon. We have better use for
that memory since current versions of Synapse, the Matrix homeserver, can be
memory hungry.

```sh
snap remove oracle-cloud-agent
apt purge snapd open-iscsi lxd lxcfs
```

## Tune server (optional)

I suggest enabling swap, since there's only 1 GB of RAM.

```sh
dd if=/dev/zero of=/swap bs=1M count=1k
chmod 0600 /swap
mkswap /swap
swapon /swap
echo '/swap none swap sw 0 0' >> /etc/fstab
```

## Point domain at server

Assuming you're using a new domain only for this you need the following DNS
records:

- A record `$domain` pointing to `$instance_external_ip_address`
- CNAME record `matrix.$domain` pointing to `$domain`
- CNAME record `element.$domain` pointing to `$domain`


## Use matrix-docker-ansible-deploy

Follow the [guide], with some tweaks:

[guide]: https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/README.md

### Guide in a nutshell, TL;DR

```sh
git clone https://github.com/spantaleev/matrix-docker-ansible-deploy/
cd matrix-docker-ansible-deploy
mkdir inventory/host_vars/matrix.$domain
cp examples/host-vars.yml inventory/host_vars/matrix.$domain/vars.yml
cp examples/hosts inventory/hosts
$EDITOR inventory/hosts
$EDITOR inventory/host_vars/matrix.$domain/vars.yml
# you'll need to rerun setup-all and start tags again if you edit vars later
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
ansible-playbook -i inventory/hosts setup.yml --tags=self-check
ansible-playbook -i inventory/hosts setup.yml -e username=$user -e password=$pass -e admin=yes --tags=register-user
```

### Serve the base domain as well (unless you already have something serving it)

```yaml
matrix_nginx_proxy_base_domain_serving_enabled: true
```

### Disable all extras because of RAM

```yaml
matrix_mxisd_enabled: false
matrix_mailer_enabled: false
matrix_coturn_enabled: true # disable to save more RAM
```

### Configure the public IP address for coturn

```yaml
matrix_coturn_turn_external_ip_address: $instance_external_ip_address
```

### Limit rooms because of limited RAM

```yaml
matrix_synapse_configuration_extension_yaml: |
  limit_remote_rooms:
    enabled: true
    complexity: 1.0 # this limits joining complex (~large) rooms, can be
                    # increased, but larger values can require more RAM
```

## Done!

Point your browser to https://element.$domain or use [another
client](https://matrix.org/clients).

## Maintenance

Remember to keep your VM up to date!

```sh
apt update
apt upgrade
reboot # e.g. kernel,dbus,systemd updates
```

To keep the matrix services upgraded, start by reading the [maintenance docs],
but really, take a brief look at all the documentation.

[maintenance docs]: https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/maintenance-upgrading-services.md

Come join [#synapse:matrix.org](https://matrix.to/#/#synapse:matrix.org) to
discuss homeservers. Or say hi to
[@ptman:ptman.name](https://matrix.to/#/@ptman:ptman.name), who wrote this
guide (also <a
href="mailto:paul.totterman@gmail.com">paul.totterman@gmail.com</a>).

Please contribute improvements to this guide via PRs ([github] or [gitlab])

[github]: https://github.com/ptman/matrix-docs
[gitlab]: https://gitlab.com/ptman/matrix-docs
