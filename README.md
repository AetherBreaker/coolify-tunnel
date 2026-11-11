# coolify-tunnel

Lets Coolify on the VPS deploy to devices behind NAT, which it can't reach directly. Each device keeps an outbound SSH connection to this container (port **2223**) and opens one reverse forward to its own SSH server. Coolify then reaches the device at `<hostname>-tunnel:<port>` over the `coolify` network. Coolify allows only one server per address, whatever the port, so each device gets its own name, a network alias of this container (see [Device names](#device-names)). Nothing is configured on the VPS host.

It's a forwarding-only `sshd`. A device's key may open only its own port: no shell, no local forwards, no other port.

## Deploying

1. **Make the host key** once, on any machine, and keep it: every device pins its fingerprint.

   ```powershell
   ssh-keygen -t ed25519 -N '""' -C coolify-tunnel -f coolify-tunnel.key
   [Convert]::ToBase64String([IO.File]::ReadAllBytes("coolify-tunnel.key")) | Set-Clipboard
   ```

   The repository ignores `coolify-tunnel.key` and `coolify-tunnel.pub` in its root.
2. **Create the application in Coolify:** a Docker Compose application from this repository, branch `main`, compose file `docker/compose.yaml`. Add the clipboard's contents as the secret `TUNNEL_SSH_HOST_KEY_B64`, then deploy.
3. **The log** prints `coolify-tunnel host key: 256 SHA256:...` at every start. Devices compare against it when they set up their tunnel.

Port 2223 is published by Docker, which bypasses `ufw` on the VPS. A firewall in the hosting provider's panel would still need to allow it.

## Adding a device

On the device, as root:

```sh
ssh-keygen -t ed25519 -N '' -C "$(hostname)-tunnel" -f /etc/coolify-tunnel/key
ssh-keyscan -t ed25519 -p 2223 <vps> > /etc/coolify-tunnel/known_hosts   # check the fingerprint against the log
```

…and a systemd service that runs, with `Restart=always`:

```sh
ssh -N -p 2223 -i /etc/coolify-tunnel/key -o UserKnownHostsFile=/etc/coolify-tunnel/known_hosts \
  -o StrictHostKeyChecking=yes -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 \
  -R 0.0.0.0:<port>:localhost:22 tunnel@<vps>
```

(The timeclock test image's `timeclock-tunnel-setup` does all of this.)

Then:

1. Add the device's line to [`docker/authorized_keys`](docker/authorized_keys), with a comment naming the device and its project, and `<hostname>-tunnel` to the aliases in [`docker/compose.yaml`](docker/compose.yaml). The next free port starts at 22001.

   ```text
   restrict,port-forwarding,permitlisten="0.0.0.0:<port>" ssh-ed25519 AAAA... <hostname>-tunnel
   ```

2. Commit, push, and redeploy in Coolify.
3. In Coolify, add a server at `<hostname>-tunnel`, port `<port>`, user `root`, with a key whose public half is in the device's `/root/.ssh/authorized_keys`.

## Device names

Each `<hostname>-tunnel` is a network alias of this container on the `coolify` network, so Docker's DNS resolves it for
Coolify. Until 2026-11-11 an ssh rule in Coolify's container (`Host *-tunnel` / `HostName coolify-tunnel`, mounted from
`/data/coolify/ssh_config.d/` by `/data/coolify/source/docker-compose.custom.yml`) mapped every such name here instead;
Coolify 4.7.5 runs ssh with `-F /dev/null`, which skips it, and every device deploy failed with "Could not resolve
hostname". The rule is still mounted, unused.
