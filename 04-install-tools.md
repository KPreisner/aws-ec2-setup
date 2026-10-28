# 4. Install tools

On the instance.

## Everyday packages

```
sudo apt-get install -y git curl wget ca-certificates gnupg rsync jq unzip make zip htop tmux tree build-essential
```

`tmux` matters on a remote box: start long jobs inside it (`tmux new -s build`) so they survive a
dropped SSH connection. Reattach with `tmux attach -t build`.

## Docker Engine and Compose

Install from Docker's own apt repository. Ubuntu's `docker.io` package lags behind and has no
Compose v2 plugin.

```
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Let `ubuntu` run Docker without `sudo`, and cap container log sizes so logs can't fill the disk:

```
sudo usermod -aG docker ubuntu
echo '{ "log-driver": "local", "log-opts": { "max-size": "20m", "max-file": "5" } }' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
exit
```

Reconnect so the group change applies, then check:

```
docker run --rm hello-world
docker compose version
```

Being in the `docker` group is effectively root on the box. That's fine for a single-user
instance.

## Task (go-task)

For projects with a `Taskfile.yml`:

```
sudo sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin
task --version
```

## uv and Python

uv installs and manages Python itself, so there's no need for `apt install python3-...`.

```
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env
uv --version
```

Install the latest Python and make it the `python` / `python3` on your PATH:

```
uv python install --default
python3 --version
```

This puts the new Python in `~/.local/bin`, ahead of Ubuntu's `/usr/bin/python3`. Leave the
system one alone: apt and other OS tools depend on it, and `sudo` still uses it. In projects, use
`uv init` / `uv add` / `uv run`, and pin a version per project with `uv python pin 3.x` if needed.

Upgrade later with `uv self update` and `uv python upgrade`.

## AWS CLI

```
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install
rm -rf /tmp/aws /tmp/awscliv2.zip
aws --version
```

On Graviton (arm64) instances, use `awscli-exe-linux-aarch64.zip` instead.

It installs now, but AWS commands will fail with "Unable to locate credentials" until the instance
has an IAM role (e.g. for S3). Once it does, the CLI picks up the role's credentials
automatically, so there's nothing to configure. Never put access keys on the instance.

## Claude Code

```
curl -fsSL https://claude.ai/install.sh | bash
claude --version
```

Run `claude` inside a project folder. The first run asks you to log in: it prints a URL, you open
it in the browser on the Mac, and paste the code back into the terminal. It keeps itself updated.

Run long sessions inside `tmux` so they survive a dropped connection.

## Git and GitHub

```
git config --global user.name "<your name>"
git config --global user.email "<your email>"
git config --global init.defaultBranch main
```

The `<key-name>.pem` on the Mac only logs you **into the instance**. GitHub needs its own key, made
on the box:

```
ssh-keygen -t ed25519 -C "<host>" -f ~/.ssh/github
cat ~/.ssh/github.pub
printf 'Host github.com\n  IdentityFile ~/.ssh/github\n  IdentitiesOnly yes\n' >> ~/.ssh/config
```

Copy the printed public key into **GitHub → Settings → SSH and GPG keys → New SSH key** (Title
`<host>`, Key type Authentication). Then test:

```
ssh -T git@github.com
```

Answer `yes` to the host-key prompt; it should greet you by username. Clone with the SSH URL from
now on (`git clone git@github.com:<github-user>/<repo>.git`). For a repo already cloned over HTTPS:
`git remote set-url origin git@github.com:<github-user>/<repo>.git`.

- This key can push to every repo your account can. A passphrase (asked by `ssh-keygen`) protects
  it if the disk or a snapshot leaks. Enter it once per login with `eval "$(ssh-agent -s)"; ssh-add ~/.ssh/github`.
- To limit the box to a single repo instead, add the same public key as a **deploy key** on that
  repo (repo → Settings → Deploy keys; tick "Allow write access" to push).
- If the box is ever shared or compromised, delete the key in GitHub settings; nothing else needs
  rotating.
- Avoid `ssh -A` (agent forwarding): anyone with root on the box can use your forwarded agent while
  you're connected.
