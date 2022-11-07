# Lab ZeroTier Configuration and Automation

This repository contains the configuration and automation to deploy ZeroTier via containers as a SystemD Service running on a number of hosts.  You can even deploy multiple instances of the ZeroTier container on a single host, technically - though it's not liable to work, port bindings and all.  The same ZeroTier container can connect to multiple ZeroTier networks.

## Host Requirements

- Podman
- Udica
- FirewallD

The Ansible Playbook will install all the latest versions of these packages on the target hosts.


## Using this repository

First off, the configuration in this repo is mine which means it's not likely to be usable by you - and you don't have edit access to this repo more than likely.  So, you'll need to fork this repo and then clone your forked repo to your local machine.

1. [Fork this repo](https://github.com/kenmoini/lab-zerotier/fork)
2. Clone your forked repo to your local machine - `git clone https://github.com/YOUR_USERNAME/lab-zerotier.git`.
3. If you'll be running the Playbook manually with the CLI then set up your inventory file - see the [examples/inventory](examples/inventory) file for an example of how to do this.  If you plan on using this with Ansible Tower you can skip creating the inventory file.
4. Now create your site configuration variable file - you can find the example file in the [examples/site_config.yml](examples/site_config.yml) file.  You'll need to edit this file to match your inventory, the services running on each host, and the ZeroTier networks those services will join.  Write your customized version to `vars/site_config.yml`.
5. Now you can run the Playbook to deploy the ZeroTier services to your hosts.  If you're using the CLI, you can run the Playbook like this:

```bash=
ansible-playbook -i inventory deploy.yml
```

> If you're using Ansible Tower or the AAP2 Controller, continue to the next step.


## Setting Up Ansible Tower for GitOps-y goodness

Ideally as soon as you commit a change to the `main` branch of this repo, the changes will be synced to the actual services running on the hosts.  To do this, we'll use Ansible Tower to run the `deploy.yml` playbook.  Assuming you already have Ansible Tower or the AAP2 Controller installed and ready to go, you can follow these steps to set things up:

1. Create a **Credential(s)** for the systems that you will be connecting to.  You can use the `Machine` credential type for this.  You can also use the `Source Control` credential type for the Git repo that you'll be using for the configuration files.
2. Create an **Inventory** for the hosts that you'll be deploying the DNS services to.
3. Add the **Hosts** to the **Inventory**.
4. Create a **Project** for your fork of this Git repo.  I like to keep the names the same as the repo, so I named mine `lab-zerotier`.  Make sure to check the `Update Revision on Launch` box.
5. Create a **Template** for the `deploy.yml` Playbook, add the Credentials you'll need to connect to the hosts.  Important: Make sure to check the `Enable Webhook` box.  This will allow you to trigger the Playbook from a GitHub/GitLab webhook.
6. Once the Template is created, you can get the `Webhook URL` and the `Webhook Key` from the **Details** page of the Template.  You'll need these to set up the Webhook in GitHub/GitLab.

Assuming you'll be using this with GitHub, you can follow these steps to set up the Webhook: https://docs.ansible.com/ansible-tower/latest/html/userguide/webhooks.html

> With this, now when you perform a `git push` to the repo to update your ZeroTier configuration, the Playbook will be triggered and the changes will be automatically deployed to the hosts!

---

## Manual Fedora/RHEL Setup

Here are some instructions for setting up a Fedora/RHEL system manually. This is not the recommended way to install the software, but it is useful for debugging.

```bash
dnf update -y

dnf install firewalld podman udica -y

systemctl enable --now firewalld

firewall-cmd --add-port=9993/tcp --add-port=9993/udp --permanent
firewall-cmd --reload

mkdir -p /var/lib/zerotier-one

# SElinux Stuff
semanage boolean -m --on nis_enabled
semanage boolean -m --on virt_sandbox_use_all_caps
semanage boolean -m --on virt_use_nfs
setsebool -P mmap_low_allowed 1
semanage port -a -t http_port_t -p tcp 9993

semodule -i zerotier.cil /usr/share/udica/templates/base_container.cil
semodule -R

systemctl enable --now podman

podman create --name zerotier \
 --cap-add=NET_ADMIN --cap-add=SYS_ADMIN \
 --net=host --device /dev/net/tun \
 -v /var/lib/zerotier-one:/var/lib/zerotier-one:z \
 --security-opt label=type:zerotier.process \
 --healthcheck-command 'CMD-SHELL bash /healthcheck.sh' \
 --healthcheck-interval=30s \
 docker.io/zerotier/zerotier:latest \
 <SPACE SEPARATED ZEROTIER NETWORK IDS HERE>

podman start zerotier

podman exec zerotier zerotier-cli info
podman exec zerotier zerotier-cli listnetworks

# If this node is routing
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-ipv4forward.conf
sysctl -p /etc/sysctl.d/99-ipv4forward.conf

# If this node is a bridge
ZT_IFACE=$(ip a | grep ': zt' | cut -d ' ' -f 2 | sed 's/://g')
BRIDGE_IFACE="bridge0"
BRIDGE_ZONE=$(firewall-cmd --get-active-zones | grep bridge0 -B 1 | head -n 1)
TARGET_ZONE="internal"

## Add the Ifaces to the internal zone
firewall-cmd --change-zone=$ZT_IFACE --zone=$TARGET_ZONE
firewall-cmd --change-zone=$BRIDGE_IFACE --zone=$TARGET_ZONE

## Add Masquerading to the Zone
firewall-cmd --zone=$TARGET_ZONE --add-masquerade

## Enable intra-zone forwarding
firewall-cmd --zone=$TARGET_ZONE --add-forward

## Add cockpit to internal zone, optional
firewall-cmd --zone=$TARGET_ZONE --add-service=cockpit

## Make the changes permanent
firewall-cmd --runtime-to-permanent
```

---

## RHEL UBI-based Container

In the `container/` directory you can find the needed items to build a container to run ZeroTier that is based on the Red Hat Enterprise Linux Universal Base Image.  It should work, but is untested in full deployment, so use at your own risk.  The official ZeroTier container is consumed and updated more easily.
