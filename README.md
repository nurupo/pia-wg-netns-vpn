# WireGuard VPN in a Network Namespace

A script for setting up and managing a WireGuard VPN in a network namespace.

Supports Private Internet Access (PIA) VPN only, but can be (relatively) easily modified to work with any other VPN provider as the PIA bits are not that tightly integrated with the rest of the script.

This is just something I wrote for myself that I'm sharing on the off chance someone finds it useful.
I have no plans on supporting any other VPN provider for now, but if in the future I change which VPN service I use, I might update the script and rename the repo appropriately.

## Why

### Why run a VPN in a network namespace?

By running a VPN in a network namespace you get these awesome perks for free:

  - Ability to run only selected applications through VPN, instead of running everything on your computer through it.
  - Kill-switch functionality, as the WireGuard VPN is the only network interface in the network namespace, aside from the loopback, so if it goes down -- there is no network in that namespace.
  - DNS protection, as applications inside the namespace will use VPN's DNS servers by default.
  - Ability to nest VPN connections. Who doesn't want to be behind 7 proxies?

To get these features in traditional, non-namespace, VPN setups you would have to create a GID per VPN, setup iptables/nftables mangle rules to fwmark application traffic based on GID, setup masquerade nat rules, add a routing table for the VPN traffic, setup routing rules for the fwmark, etc. -- so much work and potential to get things wrong.

### Why not just use \<this-other-script\>?

You might be wondering why I don't just use [`pia-foss/manual-connections`](https://github.com/pia-foss/manual-connections) or something else.

The issue is that all these scripts use [`wg-quick`](https://manpages.debian.org/unstable/wireguard-tools/wg-quick.8.en.html), which doesn't support creating WireGuard connections in a separate namespace and does a lot of unnecessary for us stuff.

To create a WireGuard connection in a separate namespace, [you want to create the wg interface in the init namespace and then move it into a separate namespace, that way it remembers that it was created in the init namespace and routes traffic though it](https://www.wireguard.com/netns/).

# Setup

```bash
# install the scripts
sudo install -o root -g root -m 755 -D vpn /usr/local/bin/vpn
sudo install -o root -g root -m 755 -D bash-completion/completions/vpn /usr/local/share/bash-completion/completions/vpn
# install deps (Debian/Ubuntu)
# bash-completion is optional but highly recommended
# sudo is required if running the script as non-root user
sudo apt install bash-completion curl jq sudo wireguard-tools
# this will create directory structure under /var/lib/vpn and prompt about missing deps
vpn regions
# setup other things
sudo install -o root -g root -m 600 pia_ca.rsa.4096.crt /var/lib/vpn/secret/pia_ca.rsa.4096.crt
sudo install -o root -g root -m 600 pia_auth.sh /var/lib/vpn/secret/pia_auth.sh
# enter your username/password
sudo editor /var/lib/vpn/secret/pia_auth.sh
```

By default the script uses `/var/lib/vpn` for its purposes.
You can change the location by changing the `VPN_ROOT_DIR` variable in the script.
Note that `VPN_ROOT_DIR` will be chown'ed to root to protect username/password whenever the script runs.

# Usage

```bash
$ vpn --help
vpn - creates a network namespace with a PIA WireGuard VPN connection in it.

Usage: [sudo   ] vpn regions                        list available VPN regions
       [sudo   ] vpn up <name> <region-id>          create VPN <name> connected to <region-id> region
       [sudo   ] vpn down <name>                    delete VPN <name>
       [sudo   ] vpn list                           list created VPNs
       [sudo   ] vpn stat <name>                    show information for VPN <name>
       [sudo -E] vpn exec <name> <program> [args]   run a program in VPN <name>'s network namespace as the current user

It's preferred that you don't call the script with "sudo", the script will
auto-sudo with the correct flags on its own.

A bit counter-intuitively, "sudo -E vpn exec" will run the program as the user
who invoked sudo, not necessarily as the root user. Run it as "sudo sudo vpn
exec" if you want to run it as root.
```

## FAQ

### How to access a web server running in a network namespace / how to forward a port?

You can use the provided `netns-socat-forward.service` as an example of how to forward connections made on host's 127.0.0.1:1234 to 127.0.0.1:1234 inside the network namespace.

Edit `netns-socat-forward.service` for your needs: host ip port, netns ip port, tcp or udp, vpn name, rename the file, change syslog identifier, etc., and run:

```bash
sudo install -o root -g root -m 644 netns-socat-forward.service /etc/systemd/system/netns-socat-forward.service
sudo systemctl daemon-reload
sudo systemctl start netns-socat-forward.service
sudo systemctl status netns-socat-forward.service
```

### How to make a desktop shortcut run a program in a VPN?

Because `vpn exec` prompts for the sudo password, you can't simply use it in a desktop shortcut as you wouldn't get the password prompt.
One way to get such a prompt is to use `pkexec`, however it doesn't preserve environment variables, which is needed in order to run GUI applications.

This is where the `pkexec-E` helper script comes in handy -- it makes `pkexec` preserve the environment as if `sudo -E` was called.

Install the helper script:

```bash
sudo install -o root -g root -m 755 pkexec-E /usr/local/bin/pkexec-E
```

Then, if your desktop shortcut runs, for example, `app %U`, you can modify it to run the following instead:

```
/usr/local/bin/pkexec-E /usr/local/bin/vpn exec vpn-name app %U
```

That assumes that the VPN is already up, otherwise `vpn exec` would error and the `app` wouldn't run.

If you want to also bring the VPN up at the same time, just in case it's not up already, you can use the following instead:

```
/usr/local/bin/pkexec-E /bin/sh -c '/usr/local/bin/vpn up vpn-name region-id > /dev/null ; /usr/local/bin/vpn exec vpn-name app "$@"' -- %U
```

`> /dev/null` is needed because there is no tty when the vpn script is run, making echo commands fail to write, killing the script.

### How to get behind 7 proxies?

You can do so like this:

```bash
/bin/bash
vpn up vpn1 region1 && trap "vpn down vpn1" EXIT && vpn exec vpn1 /bin/bash
# we are inside vpn1's namespce
vpn up vpn2 region2 && trap "vpn down vpn2" EXIT && vpn exec vpn2 /bin/bash
# we are inside vpn2's namespce
vpn up vpn3 region3 && trap "vpn down vpn3" EXIT && vpn exec vpn3 /bin/bash
# we are inside vpn3's namespce
vpn up vpn4 region4 && trap "vpn down vpn4" EXIT && vpn exec vpn4 /bin/bash
# we are inside vpn4's namespce
vpn up vpn5 region5 && trap "vpn down vpn5" EXIT && vpn exec vpn5 /bin/bash
# we are inside vpn5's namespce
vpn up vpn6 region6 && trap "vpn down vpn6" EXIT && vpn exec vpn6 /bin/bash
# we are inside vpn6's namespce
vpn up vpn7 region7 && trap "vpn down vpn7" EXIT && vpn exec vpn7 /bin/bash
# we are inside vpn7's namespce
# do your stuff here
```

Then, after you are done, just `logout` 8 times, to get out of each bash session and trigger the set traps.

Note that when nesting past a few VPNs the connection becomes rather unstable, you might lose connection way before you get behind 7 proxies, so you might want to limit yourself it to just a few of them.

Also note that the following will NOT work [because of this](https://serverfault.com/a/961592), i.e. you need to keep a process in a network namespace for the nested namespaces to work, like what all these bash processes in the snippet above did:

```bash
# you might think this would work but it doesn't
vpn up vpn1 region1
vpn exec vpn1 vpn up vpn2 region2
vpn exec vpn2 vpn up vpn3 region3
vpn exec vpn3 vpn up vpn4 region4
vpn exec vpn4 vpn up vpn5 region5
vpn exec vpn5 vpn up vpn6 region6
vpn exec vpn6 vpn up vpn7 region7
vpn exec vpn7 /bin/bash
```

# License

GPL-3.0-only
