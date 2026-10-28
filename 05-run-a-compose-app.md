# 5. Run a Compose app

The general pattern for a Docker Compose app. Your app's own docs fill in the specifics: its
data, its config, and any one-time setup.

## Code and data

Code comes from git. Keep data under `/data` (the data volume), not inside the clone, so it
survives a fresh clone and can move to another instance with the volume.

On the instance:

```
cd ~
git clone git@github.com:<github-user>/<repo>.git
mkdir -p /data/<app>
```

## Config

Point the app at the data volume and set its secrets in an untracked file such as `.env`. Never
commit them:

```
cd ~/<repo>
echo '<APP>_DATA_DIR=/data/<app>' >> .env
```

Directories that containers write to must exist before `up`, and be owned by the uid the
containers run as. Otherwise Docker creates them as root and the app can't write. Many images run
as uid 1000, which is `ubuntu`, so `mkdir -p /data/<app>/<dir>` as `ubuntu` is enough.

## Build and start

```
tmux new -s stack
docker compose up -d --build
```

(or the project's own command, e.g. `task up`). Detach with `Ctrl-b d`, reattach with
`tmux attach -t stack`.

Images built on an Apple Silicon Mac are arm64 and won't run on an x86 instance, so build on the
box (as above), or later have CI build `linux/amd64` images and push them to a registry (ECR or
GHCR).

A fresh database doesn't know about data you copied in, so run any one-time setup the app needs
(migrations, seeding, registering datasets) once the stack is up.

## Reach the UI: SSH tunnel

Apps that publish on `127.0.0.1` (e.g. `127.0.0.1:3000` in `compose.yaml`) aren't reachable from
the internet, which is what you want until TLS and auth are sorted. Tunnel the port to the Mac:

```
ssh -N -L 3000:localhost:3000 <host>
```

Then open http://localhost:3000 on the Mac. `-N` just holds the tunnel; `Ctrl-C` closes it. VS
Code's **Ports** panel does the same with one click (step 2).

## Size the box from real numbers

While the app does its heaviest work, watch memory and CPU:

```
docker stats
free -h
```

If memory sits near the instance's RAM or swap use climbs, it needs a bigger instance or fewer
worker processes. Workers that each load a large model or dataset multiply memory use.

## Resize for a big job

For a job that needs more than 32 GiB (or just more cores), move up to m7i-flex.4xlarge for the
day, then back. From the Mac:

```
aws ec2 stop-instances --instance-ids <instance-id>
aws ec2 wait instance-stopped --instance-ids <instance-id>
aws ec2 modify-instance-attribute --instance-id <instance-id> --instance-type m7i-flex.4xlarge
aws ec2 start-instances --instance-ids <instance-id>
```

Afterwards, the same four commands with `m7i-flex.2xlarge`. Or in the console: stop →
**Actions → Instance settings → Change instance type** → start. The disks, the Elastic IP, and
everything on them stay; bring the stack back up afterwards. Only change between types of the
same architecture (x86 ↔ x86).

## Going public (later)

Not until the app is safe to expose: TLS, authentication, and no admin or internal endpoints
reachable from outside. Then:

1. Add a DNS `A` record (e.g. `<app>.<your-domain>` → the Elastic IP) at the DNS provider.
2. Open ports 80 and 443 with a separate `web-public` security group (below).
3. Put a reverse proxy with TLS in front (nginx + certbot, or Caddy).

### A `web-public` security group

Keep web access in its own group rather than adding rules to `ssh-from-home`. An instance can have
several groups and gets the rules of all of them, so taking the app offline again is just
detaching this one.

1. **EC2 → Network & Security → Security Groups → Create security group.**
   - **Name:** `web-public`. **Description:** e.g. `HTTP and HTTPS from anywhere`.
   - **VPC:** the same one as the instance (the default VPC).
2. **Inbound rules → Add rule**, four times:

   | Type | Source |
   |---|---|
   | HTTP (80) | Anywhere-IPv4 (`0.0.0.0/0`) |
   | HTTP (80) | Anywhere-IPv6 (`::/0`) |
   | HTTPS (443) | Anywhere-IPv4 (`0.0.0.0/0`) |
   | HTTPS (443) | Anywhere-IPv6 (`::/0`) |

   Port 80 is needed even with TLS: certbot and Caddy use it to get certificates, and it lets the
   proxy redirect `http://` to `https://`. The IPv6 rules only matter if the instance has an IPv6
   address; they're harmless otherwise.
3. Leave **Outbound rules** at the default (all traffic) → **Create security group**.
4. Attach it: **EC2 → Instances → select → Actions → Security → Change security groups** → find
   `web-public` → **Add security group** → **Save**. Keep `ssh-from-home` attached.

Changes apply immediately, with no restart. Never add SSH (22) or the app's internal ports (e.g. a
database or `3000`) to this group: only the proxy should be reachable from the internet.

### Test it with a sample page

With `web-public` attached, check the path end to end before wiring up the real app. On the
instance:

**HTTP.** The official `nginx` image serves a "Welcome to nginx!" page:

```
docker run -d --name hello -p 80:80 nginx:alpine
```

Open `http://<elastic-ip>` in a browser. Then remove it: `docker rm -f hello`.

**HTTPS.** Caddy, as the proxy in front, gets a free Let's Encrypt certificate for the domain on
its own, renews it, and redirects HTTP to HTTPS. The DNS `A` record must already point at the
Elastic IP (check with `dig +short <app>.<your-domain>`), since Let's Encrypt connects to it
there on port 80.

```
docker network create hello
docker run -d --name hello --network hello nginx:alpine
docker run -d --name caddy --network hello -p 80:80 -p 443:443 -v caddy_data:/data \
  caddy:2 caddy reverse-proxy --from <app>.<your-domain> --to hello:80
docker logs -f caddy
```

Wait for `certificate obtained successfully` in the logs (`Ctrl-C` stops following), then open
`https://<app>.<your-domain>`. If it fails, the logs say why; usually DNS isn't pointing at the
box yet or port 80 isn't open.

Clean up:

```
docker rm -f caddy hello
docker network rm hello
```

Keep the `caddy_data` volume if you'll use Caddy for the real app: it holds the certificate, and
Let's Encrypt limits how often you can get a new one for the same domain. For the real app, swap
`hello:80` for the app's container and port, usually as a `caddy` service in its `compose.yaml`.
