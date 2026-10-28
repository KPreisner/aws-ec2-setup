# 2. Fixed IP and SSH

## Elastic IP

The auto-assigned public IP changes every time the instance stops and starts. An Elastic IP stays
put, which matters for `~/.ssh/config` and, later, a DNS record (e.g. `<app>.<your-domain>`).

1. **EC2 → Network & Security → Elastic IPs → Allocate Elastic IP address** → Allocate.
2. Select it → **Actions → Associate Elastic IP address** → Resource type Instance → pick the
   instance → Associate.
3. Optional: give it a Name tag matching the instance.

Every public IPv4 address costs about $0.005/hour (~$3.60/month), attached or not. Release the
Elastic IP when you delete the instance (step 6), or it keeps billing.

## Key file

Move the downloaded key into `~/.ssh` and lock it down; ssh refuses keys other users can read:

```
mv ~/Downloads/<key-name>.pem ~/.ssh/
chmod 400 ~/.ssh/<key-name>.pem
```

## SSH config

Add a host entry to `~/.ssh/config` on the Mac (use the Elastic IP):

```
Host <host>
  HostName <elastic-ip>
  User ubuntu
  IdentityFile ~/.ssh/<key-name>.pem
  IdentitiesOnly yes
  ServerAliveInterval 60
```

Then connect:

```
ssh <host>
```

Answer `yes` to the host-key prompt the first time. Ubuntu AMIs log in as `ubuntu`, which has
passwordless `sudo`.

## When your home IP changes

SSH starts timing out. Fix the rule: **EC2 → Security Groups → `ssh-from-home` → Edit inbound
rules** → on the SSH rule, Source → **My IP** → Save. Delete the old IP rather than adding a second
one.

## Copying files

`rsync` works through the same host alias:

```
rsync -avh --progress ./local-dir/ <host>:/data/local-dir/
```

The trailing slashes matter: `dir/` copies the contents of `dir`.

## VS Code Remote-SSH

VS Code reads the same `~/.ssh/config`, so nothing extra is needed on the instance.

1. Install the **Remote - SSH** extension (Microsoft) on the Mac.
2. **Cmd+Shift+P → Remote-SSH: Connect to Host… → `<host>`**. Pick **Linux** if asked. The first
   connect installs the VS Code server into `~/.vscode-server` on the instance (about a minute).
3. **File → Open Folder** → e.g. `~/<repo>`.

- Extensions that work on code (Python, Docker, Claude Code, …) install on the remote side: open
  the Extensions view and click **Install in SSH: <host>**.
- Ports a remote app listens on show up in the **Ports** panel and can be forwarded with one
  click, as an alternative to the `ssh -L` tunnel in step 5.
- VS Code forwards your Mac's SSH agent by default. The instance has its own GitHub key (step 4),
  so turn that off: Settings → search `remote.SSH.enableAgentForwarding` → untick.
- If the connection fails after your home IP changes, fix the security group rule (above).

## Alternative: Session Manager

AWS Systems Manager Session Manager gives a shell with no open port 22 and no key file, at the
cost of an IAM role on the instance and the Session Manager plugin on the Mac. Worth switching to
once the box is long-lived and public; plain SSH from one IP is fine for now.
