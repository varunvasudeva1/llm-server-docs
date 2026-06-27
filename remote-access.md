# Remote Access

Remote access refers to the ability to access your server outside of your home network. For example, when you leave the house, you aren't going to be able to access `http://<your_server_ip>`, because your network has changed from your home network to some other network (either your mobile carrier's or a local network in some other place). This means that you won't be able to access the services running on your server. There are many solutions on the web that solve this problem and we'll explore some of the easiest-to-use here.

## Table of Contents

- [Remote Access](#remote-access)
  - [Table of Contents](#table-of-contents)
  - [Tailscale](#tailscale)
    - [Installation](#installation)
    - [Exit Nodes](#exit-nodes)
    - [Local DNS](#local-dns)
    - [Third-Party VPN Integration](#third-party-vpn-integration)

## Tailscale

Tailscale is a peer-to-peer VPN service that combines many services into one. Its most common use-case is to bind many different devices of many different kinds (Windows, Linux, macOS, iOS, Android, etc.) on one virtual network. This way, all these devices can be connected to different networks but still be able to communicate with each other as if they were all on the same local network. Tailscale is not completely open source (its GUI is proprietary), but it is based on the [Wireguard](https://www.wireguard.com) VPN protocol and the remainder of the actual service is open source. Comprehensive documentation on the service can be found [here](https://tailscale.com/kb) and goes into many topics not mentioned here - I would recommend reading it to get the most of out the service.

On Tailscale, networks are referred to as tailnets. Creating and managing tailnets requires creating an account with Tailscale (an expected scenario with a VPN service) but connections are peer-to-peer and happen without any routing to Tailscale servers. This connection being based on Wireguard means 100% of your traffic is encrypted and cannot be accessed by anyone but the devices on your tailnet.

### Installation

First, create a tailnet through the Admin Console on Tailscale. Download the Tailscale app on any client you want to access your tailnet from. For Windows, macOS, iOS, and Android, the apps can be found on their respective OS app stores. After signing in, your device will be added to the tailnet.

For Linux, the steps required are as follows.

1) Install Tailscale
    ```
    curl -fsSL https://tailscale.com/install.sh | sh
    ```

2) Start the service
    ```
    sudo tailscale up
    ```

For SSH, run `sudo tailscale up --ssh`.

### Exit Nodes

An exit node allows access to a different network while still being on your tailnet. For example, you can use this to allow a server on your network to act as a tunnel for other devices. This way, you can not only access that device (by virtue of your tailnet) but also all the devices on the host network its on. This is useful to access non-Tailscale devices on a network.

To advertise a device on as an exit node, run `sudo tailscale up --advertise-exit-node`. To allow access to the local network via this device, add the `--exit-node-allow-lan-access` flag.

### Local DNS

If one of the devices on your tailnet runs a [DNS-sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole) service like [Pi-hole](https://pi-hole.net), you'll probably want other devices to use it as their DNS server. Assume this device is named `poplar`. This means every networking request made by a any device on your tailnet will send this request to `poplar`, which will in turn decide whether that request will be answered or rejected according to your Pi-hole configuration. However, since `poplar` is also one of the devices on your tailnet, it will send networking requests to itself in accordance with this rule and not to somewhere that will actually resolve the request. Thus, we don't want such devices to accept the DNS settings according to the tailnet but follow their otherwise preconfigured rules.

To reject the tailnet's DNS settings, run `sudo tailscale up --accept-dns=false`.

### Third-Party VPN Integration

Tailscale offers a [Mullvad VPN](https://mullvad.net/en) exit node add-on with their service. This add-on allows for a traditional VPN experience that will route your requests through a proxy server in some other location, effectively masking your IP and allowing the circumvention of geolocation restrictions on web services. Assigned devices can be configured from the Admin Console. Mullvad VPN has [proven their no-log policy](https://mullvad.net/en/blog/2023/4/20/mullvad-vpn-was-subject-to-a-search-warrant-customer-data-not-compromised) and offers a fixed $5/month price no matter what duration you choose to pay for.

To use a Mullvad exit on one of your devices, first find the exit node you want to use by running `sudo tailscale exit-node list`. Note the IP and run `sudo tailscale up --exit-node=<your_chosen_exit_node_ip>`.

> [!WARNING]
> Ensure the device is allowed to use the Mullvad add-on through the Admin Console first.
