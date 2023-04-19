# ec2-init

Minimalistic init script that is designed for any Linux with
OpenRC/systemd/runit to automatically bootstrap EC2 instances in the way similar
to `cloud-init` but without Python dependency (and a lot of other modules pulled
by `cloud-init`).

The script has a minimal set of modules but can effectively initialize EC2
instances: allow remote access over SSH and run custom bootstrap scripts.

## Dependencies

This shell script depends on:

-   coreutils
-   curl

Some Linux distributions could have these dependencies not installed, for
example, Void Linux requires manual installation of `curl`.

This shell script supports these init systems:

-   OpenRC
-   systemd
-   runit

## Installation

Ensure that `coreutils` and `curl` are installed. Not all distros have them
installed by default. For example, `coreutils` could be not installed if `busybox`
is installed. For example, `curl` is not installed by default on Void Linux.

### OpenRC

```shell
# install ec2-init/service file
curl -s https://raw.githubusercontent.com/sormy/ec2-init/master/ec2-init.openrc -o /etc/init.d/ec2-init
chmod +x /etc/init.d/ec2-init

# run ec2-init automatically
rc-update add ec2-init boot
```

### systemd

```shell
# install ec2-init helper script
curl -s https://raw.githubusercontent.com/sormy/ec2-init/master/ec2-init.script -o /usr/sbin/ec2-init
chmod +x /usr/sbin/ec2-init

# install service file
curl -s https://raw.githubusercontent.com/sormy/ec2-init/master/ec2-init.service -o /etc/systemd/system/ec2-init.service

# run ec2-init automatically
systemctl enable ec2-init
```

### runit

```shell
# install ec2-init helper script
curl -s https://raw.githubusercontent.com/sormy/ec2-init/master/ec2-init.script -o /usr/sbin/ec2-init
chmod +x /usr/sbin/ec2-init

# install service file
mkdir -m 755 -pv /etc/sv/ec2-init
curl -s https://raw.githubusercontent.com/sormy/ec2-init/master/ec2-init.runit -o /etc/sv/ec2-init/run
chmod 755 -v /etc/sv/ec2-init/run
ln -sfv /run/runit/supervise.ec2-init /etc/sv/ec2-init/supervise

# run ec2-init automatically
ln -sfv /etc/sv/ec2-init /etc/runit/runsvdir/default/
```

## Usage

`ec2-init` script runs on every boot, however, it doesn't reinitialize the
instance on every reboot. It initializes the instance only if instance ID from
last run doesn't match with current instance ID. So, if you take an image from
existing instance and then spawn a new instance using the same image, then
`ec2-init` will detect change in instance ID and will run init again.

The location of lock file can be change using `EC2_INIT_LOCK` option.

By default `ec2-init` runs with 3 modules `hostname`, `ssh` and `exec` but the
list of modules to execute can be changed using `EC2_INIT_MODULES` option.

The script uses `curl` to access EC2 instance metadata. It is not recommended to
change curl options, but, if needed, it can be done using `CURL` option.

The log for init process is logged to `/var/log/ec2-init.log`. The location of
log file can be changed using `EC2_INIT_LOG` option.

### Module "hostname"

Module `hostname` sets local hostname based on EC2 metadata.

In most cases needed just to differentiate hosts by shell prompt after login
into EC2 instance.

### Module "ssh"

Module `ssh` imports ssh public keys into `/root/.ssh/authorized_keys` during
boot (with default configuration).

Needed to login into EC2 instance using SSH keys provided in AWS console.

The location of target directory for `.ssh/authorized_keys` file can be changed
using `SSH_USER_HOME` option or using `SSH_USER_NAME` option.

### Module "exec"

Module `exec` runs shell script (if exists) from instance user data. It
downloads instance user data, decodes it from base64, then executes it if there
is a shebang `#!` localed on first line of user data.

Instance user data by default is downloaded and decoded to
`/var/lib/ec2-init.user-data` but the location can be changed using
`EC2_INIT_USER_DATA` option.

The output of executed shell script is also logged to `/var/log/ec2-init.log`.

Needed to programmatically initialize EC2 instance using customizable instance
provisioning shell script.

Read more about EC2 instance user data:
<https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-add-user-data.html>

## Configuration

While `ec2-init` script has sane defaults, you still have an option to change
some of them.

## OpenRC

Create file `/etc/conf.d/ec2-init` and override any variables you want.

For example:

```shell
echo 'EC2_INIT_MODULES="ssh"' > /etc/conf.d/ec2-init
```

## systemd

Edit service unit file `/etc/systemd/system/ec2-init.service`.

Add environment variables in `[Service]` section, for example:

```ini
[Unit]
Description=EC2 Init
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ec2-init
Environment="EC2_INIT_MODULES=ssh"

[Install]
WantedBy=multi-user.target
```

And then reload systemd: `systemctl daemon-reload`.

### runit

Edit service unit file `/etc/sv/ec2-init/run`.

Export any changed variables before invoking `/usr/sbin/ec2-init`:

```sh
#!/bin/sh

export EC2_INIT_MODULES=ssh

if /usr/sbin/ec2-init; then
    sv down ec2-init
fi
```

## Variables

Below is the list of all variables you can set.

| Name               | Default                      | Description                 |
| ------------------ | ---------------------------- | --------------------------- |
| EC2_INIT_LOCK      | /var/lib/ec2-init.lock       | Init lock file path.        |
| EC2_INIT_LOG       | /var/log/ec2-init.log        | Init log file path.         |
| EC2_INIT_USER_DATA | /var/lib/ec2-init.user-data  | Init user data.             |
| EC2_INIT_MODULES   | hostname ssh exec            | Modules to execute on init. |
| SSH_USER_NAME      | $(whoami)                    | User name for ssh init.     |
| SSH_USER_HOME      | autodetect using /etc/passwd | User home directory (ssh).  |
| CURL               | curl $CURL_OPTS              | CURL and options.           |

Default CURL_OPTS:
`curl --silent --max-time 1 --retry-delay 1 --retry 10 --retry-all-errors --fail`

# Cleanup

Would like to cleanup created by `ec2-init` files?

```shell
rm /var/log/ec2-init.* /var/lib/ec2-init.*
```

# License

MIT
