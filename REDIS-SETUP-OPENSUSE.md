# Redis Setup on openSUSE Tumbleweed

This guide documents the steps to install and configure Redis on openSUSE Tumbleweed for development use, including the troubleshooting required to get it working end-to-end.

## Overview

- System: openSUSE Tumbleweed
- Redis version installed: 8.10.1
- Managed via: systemd template service (`redis@.service`)
- Endpoint: `localhost:6379`

---

## Steps (in sequence)

### 1. Verify the environment

Confirm the OS and that Redis is not already installed:

```bash
cat /etc/os-release
which redis-server redis-cli
```

### 2. Find the Redis package

openSUSE ships Redis in its default repositories:

```bash
zypper search redis
```

The relevant package is simply named `redis` (Persistent key-value database).

### 3. Install Redis

```bash
sudo zypper install -y redis
```

> Note: `sudo` requires a password. Run this in your own terminal — piping credentials
> through `sudo -S` would expose the password in shell history/output.

### 4. Verify the installation

```bash
redis-server --version
redis-cli --version
```

Expected output (version may vary):

```
Redis server v=8.10.1 ...
redis-cli 8.10.1
```

### 5. Understand the service layout

openSUSE packages Redis with a **systemd template unit** rather than a plain `redis.service`:

- Unit: `/usr/lib/systemd/system/redis@.service`
- Runs as user `redis` / group `redis`
- Executes: `redis-server /etc/redis/<instance>.conf`

Because it is a template, you must create a config file and start a named instance.

### 6. Create the instance config

Copy the shipped template to the instance config (default `redis.conf` with port 6379):

```bash
sudo cp /etc/redis/redis.default.conf.template /etc/redis/redis.conf
```

### 7. Start and enable Redis on boot

```bash
sudo systemctl enable --now redis@redis
```

### 8. Verify status

```bash
sudo systemctl status redis@redis --no-pager
```

### 9. Test connectivity

```bash
redis-cli ping
```

A successful reply is:

```
PONG
```

---

## Troubleshooting

### Problem: `redis-cli ping` → Connection refused

**Symptoms:** The client cannot connect to `127.0.0.1:6379`. Nothing was listening on the port.

```bash
sudo systemctl status redis@redis --no-pager
```

**Found:** The service reported:

```
Active: failed (Result: start-limit-hit)
Process: 919 ExecStart=/usr/sbin/redis-server /etc/redis/redis.conf (code=exited, status=1/FAILURE)
```

The `start-limit-hit` result means systemd aborted after too many rapid restart attempts. The unit tried 5 times and gave up.

### Problem: `can't open config file '/etc/redis/redis.conf': Permission denied`

**Root cause investigation:**

Inspect the service logs:

```bash
sudo journalctl -u redis@redis --no-pager -n 40
```

The repeating error was:

```
redis-server: # Fatal error, can't open config file '/etc/redis/redis.conf': Permission denied
```

**Explanation:** Redis runs as the unprivileged `redis` user, but the config file created in step 6 did not grant that user read access. Because the service uses `User=redis` and the config lives under the root-only `/etc/redis/` directory, the `redis` user couldn't read it — so the server exited immediately with status 1, and systemd kept retrying until hitting the start limit.

Confirm the relevant settings in the config:

```bash
sudo grep -inE '^(port |unixsocket|bind|protected-mode|daemonize|dir |logfile)' /etc/redis/redis.conf
```

**Fix — grant the redis user read access and restart:**

```bash
# Give the redis user ownership and read permission on the config
sudo chown redis:redis /etc/redis/redis.conf
sudo chmod 640 /etc/redis/redis.conf

# Clear the failed (start-limit) state, then start fresh
sudo systemctl reset-failed redis@redis
sudo systemctl start redis@redis
```

**Verify:**

```bash
sudo systemctl status redis@redis --no-pager
redis-cli ping
```

Expected and confirmed result:

```
PONG
```

---

## Final state

Redis is up and running.

- Version: Redis 8.10.1
- Running as: systemd service `redis@redis`, enabled on boot
- Endpoint: `localhost:6379`
- Verified: `redis-cli ping` → `PONG`

## Useful commands

- Start/stop: `sudo systemctl start|stop redis@redis`
- Check status: `sudo systemctl status redis@redis`
- Live monitoring: `redis-cli monitor`
- Config: `/etc/redis/redis.conf`
