# systemd Jellyfin Container

Podman systemd unit file for running a Jellyfin media server as a systemd service, allowing users to perform advanced operations such as dependency
declaration (mounting other filesystems, waiting for network connection), reverse dependencies (ie: requiring this service to be online before running other units).
Take a look at systemd units for more information about the topic.

## Motivation

I have lots of media in my workstation, and I wanted Jellyfin to run at boot, so as long as the computer has started properly, any client would be able to stream media.
Jellyfin devs encourages users using a rootless account, with lingering enabled to run a rootless container as a service,
but what if the container could be properly isolated from the host, making this whole setup as simple as possible, like running native Jellyfin?

So I came up with this idea. By running a podman container as root and later generating a service file with `sudo podman generate systemd jellyfin --new --name --files` I could
run Jellyfin as a systemd service that runs at `multi-user.target`, allowing it to run without additional changes to the system!

Now, running as root is actually discouraged by the Jellyfin devs, as the service may be vulnerable to malware attacks, but as long as you
have configured a MAC security system (ie: SELinux or AppArmor) the attacker would be restricted to the volatile container, as the host filesystems would be read-only,
making their changes not persist a service restart.

AppArmor confines by default rootful containers with the `containers-default` profile, you should ensure this is the case by running `aa-status` and ensuring 
the binary executing inside the container is inside the **enforce** section.
SELinux should confine all containers by default, at least in RHEL/Fedora.

## Installation

Copy the file to `/etc/systemd/system/container-jellyfin.service`. Reload the systemd daemon with `systemctl reload-daemon`.

Afterwards, enable the container and run it with `systemctl start --now container-jellyfin.service`.

The web interface should be exposed at `localhost:8096`. Finish the initial configuration and everything should be up and running!

Note that the service will still be running as UID/GID 1000, despite being inside a root container. This is done for securing even more the container, and allowing the user
to copy existing config files to the volumes of the container (cache && config).

## HW transcoding

You may want to do hardware accelerated transcoding when streaming to clients. With AMD/Intel, you shouldn't require any additional setup, as the MESA drivers on Linux
provided in the container image should be able to do it without any problem.

### Nvidia HW transcoding

For Nvidia, you will need to install the `nvidia-container-toolkit` package.

[Note the current package at the AUR is broken](https://gitlab.com/nvidia/container-toolkit/container-toolkit/-/issues/17)

After you've installed it, run the nvidia-ctk command to generate a CDI specification:

```
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

Now, uncomment this line at the service file for sharing the Nvidia gpu with the container:

```
#       --device nvidia.com/gpu=all 
```
