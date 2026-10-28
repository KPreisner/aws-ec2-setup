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
2. Add inbound **HTTP 80** and **HTTPS 443** from `0.0.0.0/0` (and `::/0`) to the security group,
   or a separate `web-public` group.
3. Put a reverse proxy with TLS in front (nginx + certbot, or Caddy).
