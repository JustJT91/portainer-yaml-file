# Pi-hole + Unbound on Raspberry Pi 5: A DevOps Post-Mortem

This repository documents the deployment of a privacy-focused, recursive DNS stack using **Pi-hole** and **Unbound** via Docker on a Raspberry Pi 5. This guide serves as a "Lessons Learned" manual for navigating common containerization, architecture, and networking hurdles.

## 🏗️ Project Architecture
- **Hardware:** Raspberry Pi 5 (ARM64)
- **Deployment:** Docker Compose via Portainer
- **Primary DNS:** Pi-hole (Ad-blocking & Dashboard)
- **Upstream Resolver:** Unbound (Recursive DNS - No 3rd party providers)
- **Network Mode:** Bridge

---

## 🧠 Lessons Learned: Top 5 Infrastructure Pitfalls

### 1. Hardware Architecture Mismatch (`exec format error`)
**The Problem:** Many Docker images default to `amd64` (Intel/AMD). Attempting to run these on the Pi 5's **ARM64** processor causes the container to crash instantly.
**The Lesson:** Always verify the "Tags" on Docker Hub for ARM64 compatibility.
- **Fix:** Switched to the `mvance/unbound-rpi` image, optimized for the Raspberry Pi.

### 2. The "Volume Overwrite" Trap (`Code 127`)
**The Problem:** Mapping a host directory to a container's root folder (e.g., `/opt/unbound/`) replaces the container's internal binaries with the host's empty folder, "deleting" the program.
**The Lesson:** Use precise volume mapping to subdirectories.
- **Fix:** Mapped specifically to the config path: `/home/user/pi-dns/unbound:/opt/unbound/etc/unbound/`.

### 3. Port 53 Battles (`systemd-resolved`)
**The Problem:** Linux distributions like Ubuntu/Raspbian run an internal DNS service that occupies Port 53, blocking Pi-hole from starting.
**The Lesson:** The Host OS and the Docker Container cannot both use the same port.
- **Fix:** Disabled and stopped `systemd-resolved` on the Pi host.

### 4. Bridge Networking vs. Host Mode
**The Problem:** In `host` mode, containers compete for the Pi's actual ports.
**The Lesson:** `bridge` mode creates a private network where containers can talk to each other via internal names (e.g., `unbound:5335`) without clashing with the host OS.

### 5. The "Client-Side Leak" (IPv6 & DoH)
**The Problem:** The server was healthy, but the laptop still saw ads.
**The Lesson:** Modern browsers and OSs use **IPv6** and **Secure DNS (DNS over HTTPS)** as "backdoors" to bypass local DNS settings.
- **Fix:** Disabled IPv6 on the client machine and toggled off Chrome's "Secure DNS" feature to force traffic through the Pi-hole.

---

## 🛠️ Troubleshooting Checklist (Order of Operations)

Follow these steps in order when the stack isn't behaving:

1. **Check Container Logs:** `docker logs <container_name>` (Look for Architecture or Permission errors).
2. **Verify Architecture:** Run `uname -m` on the host to ensure it's `aarch64`.
3. **Check Port Occupation:** Run `sudo lsof -i :53` to find blocking services.
4. **Test Internal Handshake:** Run a query from inside the Pi-hole container to Unbound:
    `docker exec -it pihole dig @unbound -p 5335 google.com`
5. **Test Local Network Path:** From your PC terminal, run:
    `nslookup google.com <Pi_IP_Address>`
6. **Verify Client Configuration:** Flush DNS (`ipconfig /flushdns`) and disable IPv6.

---


```yaml
ound/etc/unbound/"
    restart: unless-stopped
