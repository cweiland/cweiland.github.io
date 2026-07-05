# wg-mesh

A single-page, client-side generator for **point-to-point WireGuard mesh** configs.

- Every link between two peers gets its own dedicated **/30** subnet.
- Every peer runs **one WireGuard interface per link** (a 3-node full mesh ⇒ 2 interfaces per node), each interface has exactly **one `[Peer]`** entry — true point-to-point, not a hub-and-spoke interface.
- Key pairs are generated **in your browser** with TweetNaCl (X25519, clamped the same way `wg genkey`/`wg pubkey` do) — nothing is sent anywhere.
- Overrides let you pin a specific private key:
  - on **one interface** of one peer (one side of one link),
  - on **all interfaces** of one peer,
  - on **both sides of one link**.
- A topology graph is drawn from whatever links are currently enabled.
- Generated configs are grouped by peer, viewable per-interface, copyable, and downloadable individually or as a `.zip` (per peer or for everyone).

## Run it locally

Just open `index.html` in a browser — no build step, no server.

## Publish on GitHub Pages

1. Create a new GitHub repository (or use an existing one).
2. Add `index.html` to the repo root (or to a `/docs` folder).
3. Push to GitHub.
4. In the repo: **Settings → Pages** → under "Build and deployment", set:
   - Source: `Deploy from a branch`
   - Branch: `main` (or whichever branch you pushed to), folder `/ (root)` or `/docs` — matching where you put `index.html`
5. Save. GitHub gives you a URL like `https://<username>.github.io/<repo>/` within a minute or two.

That's it — the page has no backend and no build pipeline, so any static host works (GitHub Pages, Netlify, a plain `python -m http.server`, etc).

## Proxmox VE 9.2+ SDN fabric export

Proxmox VE 9.2 (May 2026) added **WireGuard as a native SDN fabric protocol** — an
alternative to running this mesh's `.conf` files by hand, using Proxmox's own
`/etc/pve/sdn/fabrics.cfg`.

The sidebar's "Proxmox VE 9.2+ SDN fabric" panel turns the same mesh (peers, IPs,
ports, public keys) into the `wireguard:` / `wireguard_node:` section text
Proxmox expects there, e.g.:

```
wireguard: wgmesh
	persistent-keepalive 25

wireguard_node: node-a
	endpoint node-a.example.net
	role internal
	interfaces name=wg0,listen_port=51820,public_key=...,ip=10.10.0.1/30 name=wg1,listen_port=51821,public_key=...,ip=10.10.0.9/30
	peers node=node-b,iface=wg0,type=internal,node_iface=wg0 node=node-c,iface=wg1,type=internal,node_iface=wg0
```

Only one thing is deliberately **not** generated:

- **Private keys.** Proxmox keeps those in `/etc/pve/priv/wg-keys.cfg` and writes
  that file itself when you save an interface in the GUI. Paste this tool's
  private keys in there instead of trying to hand-edit that file.

Per-interface peer wiring (the `peers` line above) **is** generated automatically —
its shape is confirmed against Proxmox's own API schema (`PVE::RESTHandler::api_dump
('PVE::API2')`, cross-checked against a real cluster), not guessed.

This feature is very new (shipped in PVE 9.2, itself only weeks old at the time
of writing) — double-check the generated snippet against your exact Proxmox VE
version before applying it.

### Talking to the Proxmox API directly (experimental)

The same panel can also call the Proxmox API from your browser instead of
copy-pasting a snippet. Endpoints and field names below are confirmed against the
real Proxmox VE SDN Fabrics API schema:

- **Test connection** — `GET /api2/json/version`. A good way to check the
  host/token/CORS/certificate combo works before trying anything else.
- **Fetch existing fabrics** — `GET /api2/json/cluster/sdn/fabrics/fabric` (base
  path overridable in the "Fabrics API path" field). Shows the raw JSON rather
  than silently re-mapping it into the peer editor.
- **Push this mesh to a fabric** — `POST .../fabrics/fabric` with `{id, protocol,
  persistent_keepalive}` to create the fabric, then one `POST
  .../fabrics/node/{fabric_id}` per peer with `node_id`, the required `protocol`,
  an `interfaces` array, and a generated `peers` array wiring each interface to
  the matching interface on the other node — then a final `PUT
  /api2/json/cluster/sdn` to apply pending changes. **Dry run is on by
  default** — it prints the exact requests without sending them so you can
  check them first.

Auth is an API token (`user@realm!tokenid=secret`), sent as an `Authorization`
header — nothing is stored or sent anywhere except directly from your browser
to the host you type in. Two things commonly stop this from working the first
time:

- **Self-signed certificate** — open `https://your-host:8006` directly in the
  same browser once and accept the certificate warning, or the `fetch()` calls
  will fail silently.
- **CORS** — `pveproxy` generally does not send `Access-Control-Allow-Origin`
  for an arbitrary page's origin (like a `github.io` URL), so the browser may
  block the response even though the request reaches Proxmox. Serving this page
  from the same host/origin as Proxmox, or from a reverse proxy in front of it
  that adds permissive CORS headers for this purpose, avoids that. If neither is
  an option, use the generated `fabrics.cfg` snippet above instead and finish the
  wiring in the GUI — no browser networking involved.

## Layout

The page is organized into six tabs:

- **Preview** — nothing but the mesh graph. It's interactive: drag a node to
  reposition it, click a node to select it (highlights its links and opens an
  info panel below the graph with its endpoint, interface count, a per-link
  on/off toggle, "view its configs", and "remove peer"), click anywhere on an
  edge to toggle that link on/off directly. Disabled links stay visible as
  dashed grey lines so there's still something to click to re-enable them.
- **Nodes** — add/remove peers and toggle links from a plain list. Validation
  errors (duplicate name, empty name) show in a box on this tab.
- **Custom** — base CIDR/port/keepalive, and pinned key overrides. CIDR/size
  validation errors show in a box on this tab.
- **Configs** — everything generated: per-peer WireGuard configs, the zip
  downloads, and the Proxmox fabrics.cfg snippet once you generate it from
  Settings.
- **Settings** — the Proxmox fabric name/export options and the API connection
  (host, token, dry run). A status box on this tab shows the outcome of the
  last Test/Fetch/Push action; full request/response detail always goes to Logs.
- **Logs** — every Proxmox API request this page has made (or would make, in
  dry run), newest first, each with an expandable raw response/error.

## Motion

Decorative animation (the flowing dashes on active links, the ambient
background blobs, the pulsing node glow) is off by default if your OS/browser
has "reduce motion" turned on — that's a real accessibility setting, and the
most likely reason you'd open this page and see no animation at all. A small
"Animations" switch in the header lets you turn it on anyway regardless of
that system setting (and off, if you'd rather not have it) — the choice is
remembered per browser via `localStorage`.

## Interface text

The on-page copy is kept minimal — labels, placeholders, and short empty
states only. Anything more technical (how the Proxmox export/API works,
what "key overrides" means, CORS caveats) sits behind a small "i" tooltip
next to the relevant field instead of a paragraph. Hover it, or tap it on
touch devices.

## Usability & accessibility

- The mesh regenerates automatically as you add/remove peers, toggle links, or
  change overrides — no need to remember to click "Generate mesh" (the button
  is still there for after you change the base CIDR/port).
- Sidebar sections are collapsible (`<details>`), keyboard-operable, and marked
  up with visible focus rings (respecting `prefers-reduced-motion` for the
  graph's animated edges and the toast).
- Peer name field: press Enter to add instead of reaching for the button.
- Copy buttons show inline "Copied ✓" feedback in addition to the toast.
- The peer tabs use proper `tablist`/`tab`/`tabpanel` ARIA roles with
  left/right arrow-key navigation; the toast is an `aria-live="polite"` region.
- Text color palette was rechecked against WCAG AA (4.5:1) — the previous
  "faint" gray and the override badges were both under that threshold; both
  are now comfortably above it.
- API buttons show a loading state and a confirmation prompt before any
  non-dry-run write to your Proxmox host.

## Importing an existing fabric

Settings → Proxmox API now has a 4-step flow:

1. Fill in host, token, and (if needed) the fabrics API path.
2. **List WireGuard fabrics** — fetches all fabrics on the cluster and keeps
   the WireGuard ones.
3. Pick one from the dropdown that appears. A short note underneath (`GET`,
   read-only) says whether it's already a pure point-to-point mesh or not —
   and if not, why (e.g. an interface with more than one peer, a subnet wider
   than `/30`, or no peer data to read topology from).
4. **Import selected fabric** — rebuilds peers + links in the tool from each
   node's `peers` field, the source of truth for topology. Anything that
   isn't already point-to-point (a hub interface fanned out to several
   `[Peer]`s, wider subnets) is simplified to one link per peer relationship,
   since interface names/ports/subnets are always regenerated fresh rather
   than reused. Older fabrics with no `peers` data fall back to pairing up
   interfaces that share the same `/30`. This replaces whatever's currently in
   Nodes/Custom, after a confirmation — nothing on Proxmox is ever changed by
   import.

Proxmox never exposes private keys through its API (they live in
`/etc/pve/priv/wg-keys.cfg`), so import can't recover the exact keys already
in use — it reconstructs topology and endpoints, then you generate fresh
keys and push again if you want the fabric fully back in sync.

## Notes on the model

- **Subnet allocation**: link N gets the N-th `/30` out of the base CIDR (default `10.10.0.0/16`), in the order links are created. The lower-index peer of the pair gets `.1`, the other gets `.2`.
- **AllowedIPs**: set to the *other side's single /32*, not a routed supernet — this models direct point-to-point tunnels rather than a routed mesh. If you want transit routing across the mesh, add extra `AllowedIPs`/routes yourself after export.
- **Ports**: each peer's interfaces get `basePort, basePort+1, basePort+2, …` in the order that peer's links were created. Different peers can reuse the same port numbers since they run on different hosts.
- **Endpoint**: only peers with an endpoint host filled in get an `Endpoint =` line in *other* peers' configs. Peers without one can still dial out to everyone else.
- **Key priority**: link-side override > peer-wide override > auto-generated (cached per peer+link, stable across re-renders until you click "Re-roll auto-generated keys").
