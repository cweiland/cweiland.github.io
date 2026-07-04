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

## Notes on the model

- **Subnet allocation**: link N gets the N-th `/30` out of the base CIDR (default `10.10.0.0/16`), in the order links are created. The lower-index peer of the pair gets `.1`, the other gets `.2`.
- **AllowedIPs**: set to the *other side's single /32*, not a routed supernet — this models direct point-to-point tunnels rather than a routed mesh. If you want transit routing across the mesh, add extra `AllowedIPs`/routes yourself after export.
- **Ports**: each peer's interfaces get `basePort, basePort+1, basePort+2, …` in the order that peer's links were created. Different peers can reuse the same port numbers since they run on different hosts.
- **Endpoint**: only peers with an endpoint host filled in get an `Endpoint =` line in *other* peers' configs. Peers without one can still dial out to everyone else.
- **Key priority**: link-side override > peer-wide override > auto-generated (cached per peer+link, stable across re-renders until you click "Re-roll auto-generated keys").
