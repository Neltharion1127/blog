---
title: "Designing Public Ingress in a Self-Managed Environment"
date: 2026-01-26
draft: false
---

You might be thinking: “Is this just a homelab?”

And yes — technically, it is.

However, running services in a self-managed environment quickly surfaces many of the same questions that appear in larger, production systems.

## Why?

The motivation is straightforward: I needed a server that could run continuously (24/7) and host multiple long-running services, including:

- Jellyfin (media streaming)
- NAS services (storage)
- Aria2 (download management)
- Additional containerised applications running via Docker

Rather than focusing on scale, the goal was to operate a real system with real constraints: limited hardware resources, a residential network, and services that should remain available without constant manual intervention.

## How?

To support these requirements, I needed a machine capable of running multiple operating systems and isolated services. I initially chose VMware ESXi, as it is a mature and widely used hypervisor that can handle such workloads reliably.

The physical host is an older PC based on an Intel E3-1231 CPU. ESXi itself proved stable and reliable for this setup, but running a hypervisor on consumer-grade hardware exposed a practical constraint: the motherboard only provided a single built-in Ethernet interface. While this was not a throughput bottleneck for the workload, it removed the option of physically separating management traffic from public-facing services, which directly shaped later networking and ingress design decisions.

From a practical standpoint, ESXi also felt unnecessarily heavy for this particular setup. While it is an excellent hypervisor in enterprise environments, the operational overhead did not align well with the scale and goals of this home lab.

I then evaluated running Linux as a guest on Windows Server. This approach worked reasonably well, but it raised a broader question: if the host operating system is already Windows, why not leverage WSL for development and service management?

WSL offered a much more convenient development workflow. I could write code on my laptop and run services directly inside a Linux environment, avoiding the friction of native Windows toolchains—particularly for C and C++ development, where historical compatibility issues often complicate the setup.

Unfortunately, Windows Server does not officially support WSL, which made this option impractical.

I ultimately settled on Windows as the host operating system, paired with WSL2 running a systemd-enabled Linux environment. This configuration struck the best balance for my needs. Windows provided strong hardware support and an excellent remote management experience—the machine runs headless without a dedicated monitor—while WSL2 delivered a clean and familiar Linux runtime for development and containerised services.

This setup also allowed the media server to run directly on Windows, taking advantage of reliable driver support and hardware acceleration, while keeping infrastructure and application logic isolated within the Linux environment.

## Network
To ensure reliable access to the home lab regardless of upstream network conditions, I adopted ZeroTier as an overlay network. This allowed me to establish stable, secure connectivity into the private network without relying on direct inbound connectivity from the public internet. In practice, ZeroTier became the primary management and control plane for the system.

Within the local network, all physical and virtual machines are bridged directly to the home router. This design choice minimised friction and avoided unnecessary network abstraction, keeping traffic flows simple and observable. Rather than optimising for strict isolation at this layer, I prioritised debuggability and operational clarity, reserving access control and exposure decisions for higher-level ingress mechanisms.

Security boundaries are enforced primarily at the network edge rather than through internal segmentation. On the OpenWrt router, all unnecessary inbound ports are closed by default, and inbound access is limited to a small, explicitly defined set of port forwarding rules targeting the E3-1231 PC host only, including SSH and RDP (MSTSC). User-facing server like media server is not exposed to the public internet at all.Access is intentionally restricted to devices connected via the ZeroTier overlay network

No other internal services are directly exposed to the public internet. This keeps the internal network flat and easy to debug, while concentrating external exposure and access control at a single, well-defined ingress point on the router.

Given the dynamic nature of the residential IP address, a simple dynamic DNS setup is used to avoid relying on hard-coded addresses.

For a personal, self-managed environment of this scale, this approach provides a pragmatic balance between security, simplicity, and operational overhead, without introducing unnecessary complexity beyond what the use case requires.