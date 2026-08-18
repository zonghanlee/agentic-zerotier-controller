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

**Additional gotchas found during live deployment (2026-08-18):**
- ZeroUI's volume mount must point at the controller's *actual* data directory (e.g. `./data` from the controller compose file's own volume mapping), not the generic `/var/lib/zerotier-one` host path shown in early drafts — that path may not exist on the host at all if the controller's compose file bind-mounts a local `./data` dir instead. Mismatch causes ZeroUI to crash-loop on a missing `authtoken.secret`.
- If the controller and ZeroUI are defined in **separate** compose files in the same directory, they default to the same Compose project name (the directory name) and Compose treats them as one project — running `docker compose -f docker-compose.zeroui.yml down` will also tear down the controller container. Use `stop`/`rm` scoped to one service, or set distinct `-p`/`COMPOSE_PROJECT_NAME` values per file, or (preferred) use one unified compose file for both services.
- `ZT_ADDR` is not actually read by `dec0dos/zero-ui:latest` — the correct env var is `ZU_CONTROLLER_ENDPOINT`. Its default (`http://localhost:9993/`) can silently fail under Node 18, which resolves `localhost` to `::1` first; the controller may not be listening on the IPv6 loopback. Set `ZU_CONTROLLER_ENDPOINT=http://127.0.0.1:9993/` explicitly to avoid this.
- **`ufw` hardening pattern for true interface-scoped zero-trust**: `ufw allow 22/tcp` (SSH, keep open everywhere), `ufw allow 9993/udp` (ZeroTier's own P2P port, keep open everywhere), `ufw allow in on <zt-iface> to any port 4444` (app port allowed only on the ZeroTier interface — find the interface name via `zerotier-cli listnetworks` on the controller), `ufw deny 4444` (catch-all deny after the interface-specific allow — ufw evaluates top-to-bottom, so order matters), then `ufw enable`. Verified this correctly blocks the app port on both the public IP and the LAN IP while allowing it only on the ZT mesh IP.
- **The controller container must explicitly join its own network to get a ZT mesh IP** — creating a network via the REST API does not add the controller as a member (see existing gotcha above). Since ZeroUI runs in `network_mode: "host"`, it has no reachable ZT IP at all until the controller does `zerotier-cli join <nwid>` on itself and is authorized via the same REST API used for other members. Without this step, ZeroUI is only reachable at the host's public/LAN IPs — the opposite of the zero-exposure goal.
- **A single SSH connection attempt timing out right after `ufw enable` is not necessarily a lockout** — it can be a transient IPv6-then-IPv4-fallback delay from the client. Retry with `ssh -4` before concluding you're locked out; an already-open SSH session survives `ufw enable` regardless (existing connections aren't retroactively dropped).

Full step-by-step commands live in `skills/zerotier-vps-controller-genesis/SKILL.md` in this package.

## Tags
zerotier, self-hosted, controller, zeroui, zero-trust, vps, agentic

## Relations
- skill: [[../skills/zerotier-vps-controller-genesis/SKILL.md]]
