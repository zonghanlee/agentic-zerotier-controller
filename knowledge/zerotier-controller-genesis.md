# ZeroTier Controller Genesis

## WHAT
concept — self-hosting a ZeroTier network controller from scratch on a fresh Linux VPS, entirely via local REST API, with zero public web-UI exposure during setup. Live-tested and corrected 2026-08-16.

## WHO
Generic reference — no personal infrastructure details. Applies to any Linux VPS.

## WHERE
Any Linux host with Docker. No specific provider assumed.

## WHY
`my.zerotier.com` (ZeroTier's SaaS) is a hosted wrapper around the same open-source controller engine every `zerotier-one` binary ships with. Self-hosting removes the 25-node free-tier cap, keeps network definitions and member credentials private, and — because the whole bootstrap is API-driven — is naturally agent-operable: an AI agent with SSH access can run the entire flow without ever touching a browser.

## HOW

**The chicken-and-egg problem:** self-hosting a controller normally needs a web UI to create the first network, but exposing that UI before it's locked down is exactly the risk being avoided. Solved by sequencing:

1. **Confirm SSH key access works, then harden.** Test key-based login succeeds before disabling password authentication — skipping this check locks you out of the box with no recovery short of console/rescue mode.
2. **Boot the controller container.** Nothing web-facing yet.
3. **Create the network and authorize nodes via local REST API**, using the controller's own `authtoken.secret`. No dashboard, no signup form.
4. **Bring up the web UI last**, sharing the controller container's network namespace (`network_mode: "container:<controller>"`) so it's reachable exclusively over the mesh it just created, never the public interface.

**Real gotchas found during live testing (2026-08-16):**
- The `zerotier/zerotier` image doesn't include `wget` — use `curl`.
- A controller is not automatically a member of its own network — verify creation via the controller API, not `zerotier-cli listnetworks`.
- A plain Docker `ports:` mapping to a container's ZeroTier IP cannot work under normal bridge networking — the ZT-assigned IP only exists inside the ZeroTier daemon container's own network namespace. The web UI container must share that namespace directly.
- ZeroUI mounts `/var/lib/zerotier-one` read-only to read `authtoken.secret` directly, using `network_mode: "host"` to connect to `127.0.0.1:9993` on port 4444 without database requirements.
- Don't remap the controller's external port away from 9993/udp unless truly unavoidable — ZeroTier's protocol announces port 9993 internally to peers regardless of host-side NAT remap, so a remapped controller leaves real peers stuck at `REQUESTING_CONFIGURATION` indefinitely.

Full step-by-step commands live in `skills/zerotier-vps-controller-genesis/SKILL.md` in this package.

## Tags
zerotier, self-hosted, controller, zeroui, zero-trust, vps, agentic

## Relations
- skill: [[../skills/zerotier-vps-controller-genesis/SKILL.md]]
