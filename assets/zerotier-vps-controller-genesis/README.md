# Ditching the SaaS: How to Self-Host Your Own ZeroTier Controller from Day 1

> **Draft Article for thing.jym.sg**  
> *Author: q / Alfred Architecture*  
> *Target: Self-hosted networking, Zero-Trust DevOps, Homelab infrastructure*

---

## 1. The Myth vs. The Reality

Many developers and network engineers mistakenly believe that using ZeroTier requires signing up for their commercial SaaS platform at `my.zerotier.com`. 

- **The Myth:** You must rely on ZeroTier Central to create Network IDs, authorize nodes, and manage IP pools.
- **The Reality:** Every standard `zerotier-one` binary contains a full-featured, built-in network controller engine (`zerotier-one -U`). ZeroTier Central is simply ZeroTier Inc.'s hosted wrapper around that exact open-source engine.

By self-hosting your controller on a commodity $4–$10/mo VPS (e.g. Contabo, Hetzner, DigitalOcean), you achieve:
- **100% Data Sovereignty**: Network definitions and member credentials remain strictly private.
- **Zero Node Count Limits**: No 25-node free tier caps.
- **Zero Cloud SaaS Lock-in**: Immune to third-party cloud outages or license policy changes.

---

## 2. The Minimal Starter Architecture (Single VPS Setup)

A beginner or enterprise engineer can deploy a complete sovereign mesh using a **single Linux VPS** acting as both the controller and a node:

```
                  ┌───────────────────────────────────────────┐
                  │          Single $4/mo Starter VPS         │
                  │                                           │
                  │  [ ZeroUI Web UI ] (Port 4444)            │
                  │          │ (REST API)                     │
                  │          v                                │
                  │  [ Self-Hosted Controller ] (Port 9993)   │
                  │          │ (Issues Network IDs)           │
                  │          v                                │
                  │  [ Local Client Node ] (Port 29993)       │
                  └─────────────────────┬─────────────────────┘
                                        │
                                        │ (P2P Encrypted Mesh)
                  ┌─────────────────────┼─────────────────────┐
                  │                     │                     │
                  v                     v                     v
            [ Your Laptop ]     [ Your Home NAS ]     [ Cloud Worker VM ]
```

---

## 3. The "Zero-Trust Genesis Flow" (Zero Public Web Exposure)

The biggest challenge in self-hosting network controllers is the **"Chicken-and-Egg" security dilemma**: *How do you access the web admin UI to create your private VPN mesh, without exposing that web UI to public internet scanners (Shodan/Censys) during setup?*

We solve this using a 3-step zero-exposure lifecycle:

### Step 1: VPS Hardening (Genesis)
Spin up a fresh VPS and immediately disable password authentication in SSH:
```bash
sudo sed -i 's/^#\?PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### Step 2: Silent CLI Bootstrap (Pattern 2)
Boot the controller container (`zerotier-controller`) and create **Zone 1** (`qLab`) silently using local `curl` REST API calls with the container's `authtoken.secret`. No web UI container is even running yet:

```bash
TOKEN=$(docker exec zerotier-controller cat /var/lib/zerotier-one/authtoken.secret)

# Create Network Zone 1
docker exec zerotier-controller curl -s -X POST \
  -d '{"name":"qLab","private":true,"v4AssignMode":{"zt":true},"routes":[{"target":"10.246.231.0/24"}],"ipAssignmentPools":[{"ipRangeStart":"10.246.231.1","ipRangeEnd":"10.246.231.254"}]}' \
  -H "X-ZT1-Auth: $TOKEN" \
  "http://localhost:9993/controller/network"

# Join operator laptop and authorize via local API
docker exec zerotier-controller curl -s -X POST \
  -d '{"authorized":true,"ipAssignments":["10.246.231.203"]}' \
  -H "X-ZT1-Auth: $TOKEN" \
  "http://localhost:9993/controller/network/<NWID>/member/<LAPTOP_NODE_ID>"
```
*Result: Laptop and VPS are now securely linked on private IP `10.246.231.x`.*

### Step 3: Web UI Deployment & Zero Exposure Lockdown (Pattern 3)
Deploy ZeroUI (`dec0dos/zero-ui:latest`) using `network_mode: "host"` bound on port **4444**:

```yaml
services:
  zeroui:
    image: dec0dos/zero-ui:latest
    container_name: zeroui
    network_mode: "host"
    environment:
      - ZT_ADDR=127.0.0.1:9993
      - PORT=4444
```

- **Public Internet Firewall (`:4444`)**: Connection Refused (0% public exposure).
- **Over ZeroTier (`http://10.246.231.x:4444`)**: Opens cleanly in your laptop browser.

---

## 4. Bonus Skill File (`zerotier-vps-controller-genesis`)

To automate this workflow, use the companion skill file located at [`skills/zerotier-vps-controller-genesis/SKILL.md`](../../skills/zerotier-vps-controller-genesis/SKILL.md).

---

## Relations
- skill: [[../../skills/zerotier-vps-controller-genesis/SKILL.md]]
- knowledge: [[../../knowledge/zerotier-controller-genesis.md]]
