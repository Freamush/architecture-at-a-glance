# Architecture at a Glance
For the last couple of years, I have been working on a large project. We are building our own gaming server network, and my role has been mainly focused on infrastructure architecture and securing it according to zero-trust best practices.
This architecture is a hybrid cloud setup, integrating my self-hosted on-premises servers with OVH cloud.

On Premises:
It consists of 6 virtual machines running on my bare-metal Proxmox hypervisor.
<img width="277" height="186" alt="image" src="https://github.com/user-attachments/assets/f3c56c8b-ad07-48cb-9f4a-cfdb879e7e0f" />

The main idea behind this design is separation of duties and VLAN segmentation. Each VM performs a specific job, and only specific users are allowed to connect. Each VM can also only see the VLANs it's permitted to, which makes further lateral movement harder in case of a breach.

VM1 is a firewall VM running OPNsense. This provides not only firewalling but also includes an IDS/IPS that sends alerts in case of anomalies (discussed further below).

VM2.x are the gaming servers themselves with a cluster running pods (this part is still in progress).

VM3 is the backend server. VM2.x instances use it to synchronize player data across game servers (pods). VM3 itself follows a micro-service architecture powered by Docker. Backend itself was built on Spring by a teammate, so I won't go into detail here.

VM4 is a miscellaneous VM used for small supporting services we access when connecting to the infrastructure. The main thing is a Docker registry, which VM2.x and VM3 pull images from. It was also previously used as a Jenkins runner for CI/CD, now retired.

VM5 is the SIEM VM running Wazuh. All VMs mentioned above (except OPNsense) are connected as agents, so the SIEM receives metrics and security-relevant events. OPNsense also forwards IPS alerts here. The Wazuh VM is integrated with a Telegram bot, so high-severity alerts notify the team immediately.

Perimeter:
To secure the on-premises network, I implemented Tailscale as a mesh VPN. Since it uses outbound connections and relay servers, there are no public open ports on the on-prem architecture, everything is hidden and inaccessible without a Tailscale account and a trusted device on the architecture's tailnet. Tailscale forms a mesh network where every device can potentially see every other, but ACL tags let me control which devices can see which VM. VMs themselves don't communicate over the tailnet, so they use their own private network via OPNsense for internal traffic.
<img width="1550" height="99" alt="image" src="https://github.com/user-attachments/assets/e58ac3e5-0c97-4d21-9146-83c66f4956e5" />

Tailscale also provides DNS and TLS certificate issuance, so all tailnet traffic can be fully encrypted using certificates generated for internal communication.

OVH
The OVH cloud infrastructure acts as the "front door." The OVH VMs are the only nodes reachable without Tailscale, they act as reverse proxies, connected to the tailnet, forwarding traffic into the on-premises network.
