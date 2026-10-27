# Cloudflare Tunnel

Today the public hostnames are Cloudflare-proxied DNS records that point at the home connection, and the router forwards 80/443 to Traefik (10.42.20.40). The origin is therefore also reachable directly by IP, bypassing Cloudflare. IPv4 scanners do find it: Traefik has logged requests whose Host is the bare public IP.

The tunnel replaces the port forward. Two `cloudflared` pods dial out to Cloudflare's edge, and Cloudflare sends the traffic back through those connections. Once the cutover is done, the router has no open port.

```
browser ──► Cloudflare edge ◄── outbound ── cloudflared (2 pods) ──► Traefik ──► apps
```

The tunnel is remotely managed. It has a single route, `*.synergia.net.ar` → Traefik, so host routing stays in Traefik and in Git.

## 1. Create the tunnel (Cloudflare dashboard)

1. Go to Zero Trust → Networks → Tunnels → Create a tunnel → Cloudflared, and name it `synergia`.
2. Copy the token. It is the long string after `--token` in the install command the dashboard shows. Ignore the install command itself.

## 2. Store the token (SOPS)

From the repo root. The token is read without echo and never hits the shell history:

```sh
read -rs TUNNEL_TOKEN \
  && kubectl create secret generic cloudflared-tunnel -n cloudflared \
       --from-literal=token="$TUNNEL_TOKEN" --dry-run=client -o yaml \
       > bootstrap/kubernetes/apps/secrets/secret-cloudflared-tunnel.yaml \
  && unset TUNNEL_TOKEN \
  && sops -e -i bootstrap/kubernetes/apps/secrets/secret-cloudflared-tunnel.yaml
```

Then:

1. Add `- secret-cloudflared-tunnel.yaml` to `apps/secrets/kustomization.yaml`.
2. Check that the file only holds `ENC[...]` values before committing.
3. Merge this branch.

The dashboard should then show the tunnel as HEALTHY, with connections from two pods.

## 3. Add the route (dashboard → the tunnel → Public Hostname)

| Field | Value |
|---|---|
| Subdomain | `*` |
| Domain | `synergia.net.ar` |
| Service | `HTTPS` → `traefik.traefik.svc.cluster.local:443` |
| Additional settings → TLS → Origin Server Name | `synergia.net.ar` |

Why the Origin Server Name:
- **The certificate:** Traefik presents the wildcard certificate (`traefik/synergia-wildcard`), which covers `synergia.net.ar`. With this field set, cloudflared verifies that certificate, so leave "No TLS Verify" off.
- **Routing:** Traefik still routes by the Host header, which keeps the original hostname.
- **Why not plain HTTP:** the `web` entrypoint only redirects to HTTPS, so an HTTP origin would loop.

## 4. Move DNS, one host first

The route only receives traffic for names whose DNS record is a proxied CNAME to `<tunnel-id>.cfargotunnel.com`.

1. Create a `tunnel-test` CNAME pointing at the tunnel. Open `https://tunnel-test.synergia.net.ar`: Traefik's 404 page proves the whole path works.
2. Switch one low-risk host, such as `homebox`, from its A record to the CNAME. Test it, including the Authelia login.
3. Switch the rest. Per-host A records override the wildcard, so either turn each one into the CNAME, or delete it and turn the `*` record into the CNAME.
4. Delete `tunnel-test`.

## 5. Close the router

In Omada, remove the 80/443 port forward (virtual server) to 10.42.20.40. Then, from outside the LAN (for example mobile data):

- every site should still load;
- `https://<public IP>` should no longer answer.

## Follow-ups

- **Real client IPs:** Traefik sees every request as coming from inside the cluster. To fix that, trust `X-Forwarded-For` from the cloudflared pods (`forwardedHeaders.trustedIPs`). Authelia's regulation (banning after failed logins) needs it.
- **CiliumNetworkPolicy:** cloudflared needs egress only to Cloudflare's edge (7844 TCP/UDP), to DNS and to Traefik. Traefik needs ingress only from cloudflared and the LAN.
- **Alerts:** scrape the metrics on :2000 and alert when `cloudflared_tunnel_ha_connections` drops.
