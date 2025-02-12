---
description: >-
  For mainnet, it's recommended to use Cosmovisor to run your node. If you've
  not used it before, then run it during a testnet to check you can get it set
  up correctly.
cover: ../.gitbook/assets/Discord Invite (29).png
coverY: 259
---

# Setting up Cosmovisor

Setting up Cosmovisor is relatively straightforward. However, it does expect certain environment variables and folder structure to be set.

Cosmovisor allows you to download binaries ahead of time for chain upgrades, meaning that you can do zero (or close to zero) downtime chain upgrades. It's also useful if your local timezone means that a chain upgrade will fall at a bad time.

Rather than having to do stressful ops tasks late at night, it's always better if you can automate them away, and that's what Cosmovisor tries to do.

## Install

First, go and get cosmovisor (recommended approach):

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest

To install a previous version, you can specify the version after the @ sign. Note that versions older than 1.4.0 can also target a specific version, at a slightly different location:
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@v1.5.0
```

{% hint style="danger" %}
When using cosmovisor, make sure that you do not have auto download of binaries on. Ensure you have the environment variable `DAEMON_ALLOW_DOWNLOAD_BINARIES` set to `false`.
{% endhint %}

Your installation can be confirmed with:

```bash
which cosmovisor
```

This will return something like:

```bash
/home/<your-user>/go/bin/cosmovisor
```

{% hint style="info" %}
Building from source allows you to target a specific version of Cosmovisor, in case you do not want to run 1.7.0 yet.
{% endhint %}

You can also build from source; cosmovisor is in the main `cosmos-sdk` repo on Github, so you can use Git tags to target a specific version. This example uses a tag, `cosmovisor/v1.5.0`, the current more stable version as some bugs are reported on v1.7.0.
```bash
git clone https://github.com/cosmos/cosmos-sdk
cd cosmos-sdk
git checkout cosmovisor/v1.5.0
make cosmovisor
cp cosmovisor/cosmovisor $GOPATH/bin/cosmovisor
cd $HOME
```

## Add environment variables to your shell

In the `.profile` file, usually located at `~/.profile`, add:

```bash
export DAEMON_NAME=junod
export DAEMON_HOME=$HOME/.juno
```

Then source your profile to have access to these variables:

```bash
source ~/.profile
```

You can confirm success like so:

```
echo $DAEMON_NAME
```

It should return `junod`.

## Set up folder structure

Cosmovisor expects a certain folder structure:

```bash
.
├── current -> genesis or upgrades/<name>
├── genesis
│   └── bin
│       └── $DAEMON_NAME
└── upgrades
    └── <name>
        └── bin
            └── $DAEMON_NAME
```

Don't worry about `current` - that is simply a symlink used by Cosmovisor. The other folders will need setting up, but this is easy:

```bash
mkdir -p $DAEMON_HOME/cosmovisor/genesis/bin && mkdir -p $DAEMON_HOME/cosmovisor/upgrades
```

### Set up genesis binary

Cosmovisor needs to know which binary to use at genesis. We put this in `$DAEMON_HOME/cosmovisor/genesis/bin`.

First, find the location of the binary you want to use:

```bash
which junod
```

Then use the path returned to copy it to the directory Cosmovisor expects. Let's assume the previous command returned `/home/your-user/go/bin/junod`:

```bash
cp $HOME/go/bin/junod $DAEMON_HOME/cosmovisor/genesis/bin
```

```bash
cp $HOME/go/bin/junod $DAEMON_HOME/cosmovisor/genesis/bin
```

## Cosmovisor init

Post v1 versions of Cosmovisor have a command that will create the directories and copy the `junod` binary into the proper directory. To create the directories and copy the binary, run this command:

```bash
cosmovisor init $HOME/go/bin/junod
```

Once you're done, check the folder structure looks correct using a tool like `tree`.

## Set up service

Commands sent to Cosmovisor are sent to the underlying binary. For example, `cosmovisor version` is the same as typing `junod version`.

Nevertheless, just as we would manage `junod` using a process manager, we would like to make sure Cosmovisor is automatically restarted if something happens, for example an error or reboot.

First, create the service file:

```bash
sudo nano /etc/systemd/system/junod.service
```

Change the contents of the below to match your setup - `cosmovisor` is likely at `~/go/bin/cosmovisor` regardless of which installation path you took above, but it's worth checking.

**Note** `cosmovisor run start` is only for the latest versions of cosmovisor. For earlier versions that line should be:

```
ExecStart=/home/<your-user>/go/bin/cosmovisor start
```

```
[Unit]
Description=Juno Daemon (cosmovisor)
After=network-online.target

[Service]
User=<your-user>
ExecStart=/home/<your-user>/go/bin/cosmovisor run start
Restart=always
RestartSec=3
LimitNOFILE=4096
Environment="DAEMON_NAME=junod"
Environment="DAEMON_HOME=/home/<your-user>/.juno"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
```

{% hint style="info" %}
A description of what the environment variables do can be found [here](https://docs.cosmos.network/main/build/tooling/cosmovisor). Change them depending on your setup.
{% endhint %}

## Start Cosmovisor

{% hint style="warning" %}
If syncing from a snapshot, do not start Cosmovisor yet.
{% endhint %}

Finally, enable the service and start it.

```bash
sudo -S systemctl daemon-reload
sudo -S systemctl enable junod
# check config one last time before starting!
sudo systemctl start junod
```

Check it is running using:

```
sudo systemctl status junod
```

If you need to monitor the service after launch, you can view the logs using:

```bash
journalctl -fu junod
```
