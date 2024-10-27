# wg-netns

[wg-quick](https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8) with support for Linux network namespaces.
A simple Python script that implements the steps described at [wireguard.com/netns](https://www.wireguard.com/netns/#ordinary-containerization).

This version is modified by EtaoinWu to support peer-to-peer interfaces for use of dn42.

## Setup

Requirements:

- Python 3.7 or newer
- `ip` from iproute2
- `wg` from wireguard-tools
- optional: [pyyaml](https://pypi.org/project/PyYAML/) python package for configuration files in YAML format, otherwise only JSON is supported

Installation:

a) With [pipx](https://github.com/pypa/pipx).

~~~ bash
pipx install git+https://github.com/EtaoinWu/wg-netns.git@main
~~~

b) With `pip`.

~~~ bash
pip install --user git+https://github.com/EtaoinWu/wg-netns.git@main
~~~

c) As standalone script.

~~~ bash
curl -o ~/.local/bin/wg-netns https://raw.githubusercontent.com/EtaoinWu/wg-netns/main/wgnetns/main.py
chmod +x ~/.local/bin/wg-netns
~~~

## Usage

First, create a configuration profile.
JSON and YAML file formats are supported.

Minimal JSON example:

~~~ json
{
  "name": "ns-example",
  "interfaces": [
    {
      "name": "wg-example",
      "address": ["10.10.10.192/32", "fc00:dead:beef::192/128"],
      "private-key": "4bvaEZHI...",
      "peers": [
        {
          "public-key": "bELgMXGt...",
          "endpoint": "vpn.example.com:51820",
          "allowed-ips": ["0.0.0.0/0", "::/0"]
        }
      ]
    }
  ]
}
~~~

Full YAML example:

~~~ yaml
# name of the network namespace where the interface is moved into, if null the default namespace is used
name: ns-example
# namespace where the interface is initialized, if null the default namespace is used
base_netns: null
# if false, the netns itself won't be created or deleted, just the interfaces inside it
managed: true
# list of dns servers, if empty dns servers from default netns will be used
dns-server: [10.10.10.1, 10.10.10.2]
# shell hooks, e.g. to set firewall rules, two formats are supported
pre-up: echo pre-up from managed netns
post-up:
- host-namespace: true
  command: echo post-up from host netns
- host-namespace: false
  command: echo post-up from managed netns
pre-down: echo pre-down from managed netns
post-down: echo post-down from managed netns
# list of wireguard interfaces inside the netns
interfaces:
  # interface name, required
- name: wg-site-a
  # list of ip addresses, at least one entry required
  address:
  - 10.10.11.172/32
  - fc00:dead:beef:1::172/128
  # can also be set via "wg set wg-site-a $key"
  private-key: nFkQQjN+...
  # optional settings
  listen-port: 51821
  fwmark: 21
  mtu: 1420
  # list of wireguard peers
  peers:
    # public key is required
  - public-key: Kx+wpJpj...
    # optional settings
    preshared-key: 5daskLoW...
    endpoint: a.example.com:51821
    persistent-keepalive: 25
    # list of ips the peer is allowed to use, at least one entry required
    allowed-ips:
    - 10.10.11.0/24
    - fc00:dead:beef:1::/64
    # by default the networks specified in 'allowed-ips' are routed over the interface, 'routes' can be used to overwrite this behaivor
    routes:
    - 10.10.11.0/24
    - fc00:dead:beef:1::/64
- name: wg-site-b
  address:
  - 10.10.12.172/32
  - fc00:dead:beef:2::172/128
  private-key: guYPuE3X...
  listen-port: 51822
  fwmark: 22
  peers:
  - public-key: NvZMoyrg...
    preshared-key: cFQuyIX/...
    endpoint: b.example.com:51822
    persistent-keepalive: 25
    allowed-ips:
    - 10.10.12.0/24
    - fc00:dead:beef:2::/64
p2p-interfaces:
  # interface name, required
- name: wg-p2p-1
  address:
    v4: 10.10.13.172/32
    v6: fc00:dead:beef:3::172/128
  private-key: 4bvaEZHI...
  listen-port: 51823
  fwmark: 23
  peer:
    public-key: bELgMXGt...
    preshared-key: 5daskLoW...
    endpoint: vpn.example.com:51820
    persistent-keepalive: 25
    allowed-ips:
    - 10.10.13.0/24
    - fc00:dead:beef:3::/64
    address:
      v4: 10.10.13.192/32
      v6: fc00:dead:beef:3::192/128
~~~

Now it's time to setup your new network namespace and all associated wireguard interfaces.

~~~ bash
wg-netns up ./example.yaml
~~~

Profiles stored under `/etc/wireguard/` can be referenced by their name.

~~~ bash
wg-netns up example
~~~

You can verify the success with a combination of `ip` and `wg`.

~~~ bash
ip netns exec ns-example wg show
~~~

You can also spawn a shell inside the netns.

~~~ bash
ip netns exec ns-example bash -i
~~~

### Extra information

For these topics:

- systemd service
- port forwarding with socat
- wireguard with dyndns
- firefox in network namespace

Please see [dadevel](https://github.com/dadevel/wg-netns)'s upstream for more information.

### Podman Integration

A podman container can be easily attached to a network namespace created by `wg-netns`.
The example below starts a container connected to a netns named *ns-example*.

~~~ bash
podman run -it --rm --network ns:/run/netns/ns-example docker.io/library/alpine wget -q -O - https://ipinfo.io
~~~
