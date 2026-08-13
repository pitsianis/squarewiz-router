# Tailscale on a Constrained OpenWrt ARM64 Router

This guide installs a compact, self-contained Tailscale build on an OpenWrt-class ARM64 router with limited writable flash.

It assumes the router reports `aarch64` and uses OpenWrt's `procd` init system.

## 1. Confirm the target architecture

On the router:

```sh
uname -m
uname -a
cat /etc/openwrt_release
```

For an `aarch64` router, build for:

```text
GOOS=linux
GOARCH=arm64
```

## 2. Obtain a reproducible source tree

On the build machine:

```sh
git clone https://github.com/tailscale/tailscale.git
cd tailscale
git checkout <VERSION-TAG>
```

Use a release tag in place of `<VERSION-TAG>` when reproducibility matters.

## 3. Build a compact ARM64 daemon

On the build machine, from the Tailscale source tree:

```sh
GOOS=linux GOARCH=arm64 ./build_dist.sh --extra-small --box -o mybinaries/tailscaled-openwrt-box ./cmd/tailscaled
```

The `--extra-small` build was used because writable storage was constrained. The resulting `tailscaled` was about 12.4 MB; exact size varies by release.

Optionally build the regular CLI too:

```sh
GOOS=linux GOARCH=arm64 ./build_dist.sh tailscale.com/cmd/tailscale
```

Verify the daemon artifact before transfer:

```sh
file mybinaries/tailscaled-openwrt-box
sha256sum mybinaries/tailscaled-openwrt-box
```

Do not try to execute the Linux/ARM64 binary on a macOS build machine. Test it on the router.

## 4. Transfer and verify

From the build machine:

```sh
scp mybinaries/tailscaled-openwrt-box root@ROUTER_IP:/tmp/tailscaled
```

On the router:

```sh
chmod 755 /tmp/tailscaled
file /tmp/tailscaled
/tmp/tailscaled --version
sha256sum /tmp/tailscaled
```

Confirm that the SHA-256 value matches the build-machine result.

## 5. Install the daemon and CLI link

On the router:

```sh
mv /tmp/tailscaled /usr/bin/tailscaled
ln -sf /usr/bin/tailscaled /usr/bin/tailscale
ls -lh /usr/bin/tailscaled /usr/bin/tailscale
```

This installation uses a `tailscale` symlink to `tailscaled` to minimize storage. Conventional package installations often provide separate CLI and daemon files.

## 6. Confirm TUN support

Tailscale needs the Linux TUN device:

```sh
ls -l /dev/net/tun
lsmod | grep tun
```

If necessary:

```sh
modprobe tun
```

On OpenWrt, the required package is typically `kmod-tun`.

## 7. Create persistent state storage

Check whether `/var` is volatile:

```sh
ls -ld /var
```

On this type of OpenWrt setup, `/var` points to `/tmp` and is erased at reboot. Keep persistent Tailscale state on the writable overlay instead:

```sh
mkdir -p /etc/tailscale
chmod 700 /etc/tailscale
```

The persistent state file will be:

```text
/etc/tailscale/tailscaled.state
```

## 8. Add an OpenWrt service

Create `/etc/init.d/tailscale` with this content:

```sh
#!/bin/sh /etc/rc.common

START=99
STOP=10
USE_PROCD=1

PROG="/usr/bin/tailscaled"
STATE="/etc/tailscale/tailscaled.state"
SOCKET="/var/run/tailscale/tailscaled.sock"

start_service() {
    mkdir -p /var/run/tailscale
    chmod 755 /var/run/tailscale

    procd_open_instance
    procd_set_param command "$PROG" \
        --state="$STATE" \
        --socket="$SOCKET"
    procd_set_param respawn
    procd_close_instance
}
```

Then enable it:

```sh
chmod 755 /etc/init.d/tailscale
/etc/init.d/tailscale enable
```

The socket belongs under `/var/run` because it is transient runtime data. The state file must remain under `/etc/tailscale` so it survives reboot.

## 9. Start and authenticate

Start the service:

```sh
/etc/init.d/tailscale start
ps | grep '[t]ailscaled'
ls -l /var/run/tailscale/tailscaled.sock
```

Authenticate the router:

```sh
/usr/bin/tailscale up
```

Open the URL printed by the command on another device, then verify:

```sh
/usr/bin/tailscale ip
/usr/bin/tailscale status
wc -c /etc/tailscale/tailscaled.state
```

The state file should be non-empty after authentication.

## 10. Verify persistence

Before rebooting:

```sh
wc -c /etc/tailscale/tailscaled.state
```

Reboot:

```sh
reboot
```

After the router is back:

```sh
ps | grep '[t]ailscaled'
wc -c /etc/tailscale/tailscaled.state
/usr/bin/tailscale ip
```

From another Tailscale device:

```sh
ssh root@<TAILSCALE-IP>
```

If the daemon starts automatically, the state file remains non-empty, and SSH works, the installation is complete.

## Non-standard details in this setup

| Item | This OpenWrt setup | Typical Linux package installation |
|---|---|---|
| CPU target | `linux/arm64` | Depends on host architecture |
| Build | Custom `--extra-small` build | Prebuilt package or binary |
| Service manager | OpenWrt `procd` | Often systemd |
| Service definition | `/etc/init.d/tailscale` | Package-provided unit/service |
| Persistent state | `/etc/tailscale/tailscaled.state` | Package-managed state directory |
| Runtime socket | `/var/run/tailscale/tailscaled.sock` | Similar transient runtime path |
| CLI | Symlink to `tailscaled` | Often separate executable |

The critical divergence is persistent state: when `/var -> /tmp`, do not use `/var/lib/tailscale` for it. It will disappear on reboot.
