# FAAS Client Tunnel – Multi-User Reverse SSH Client

This package provides a **fully automated client** for registering Linux/WSL machines with a FAAS VM and exposing local services over HTTPS using **short-lived SSH certificates** and persistent reverse SSH tunnels.

The design follows a **multi-user isolation model**, where each tunnel runs as a dedicated Linux user under systemd supervision.

---

## Features

* Token-based SSH certificate authentication
* Dynamic remote port discovery via `/whoami`
* Persistent reverse SSH tunnel
* Automatic certificate renewal
* Automatic tunnel reconnect
* Multi-user support (one tunnel per Linux user)
* systemd-managed lifecycle
* Strict localhost binding on remote side
* No manual daemon handling

Each user can expose a local service as:

```
https://<username>.bitone.in
```

---

## System Flow

### 1. Token Provisioning

Administrator provides a **TOKEN** mapped to:

* Logical user name (subdomain)
* Allowed SSH principals
* Assigned remote port
* Maximum certificate TTL

---

### 2. Certificate Request

Client sends:

* SSH public key
* Token

To:

```
https://bitone.in/sign-cert
```

Server validates:

* Token exists
* Token is active
* Token has assigned port
* Token has non-empty name

Server returns a signed SSH certificate limited to allowed principals.

---

### 3. Port Discovery

Client queries:

```
https://bitone.in/whoami
```

Receives assigned remote port (e.g. `9004`).

---

### 4. Tunnel Establishment

Client opens:

```
local:8080 → vm:127.0.0.1:9004
```

Using:

```bash
ssh -N -R 127.0.0.1:9004:localhost:8080 principal@bitone.in
```

Remote port is always bound to `127.0.0.1` for safety.

---

### 5. Public Routing

VM maps:

```
https://<username>.bitone.in → 127.0.0.1:<assigned_port>
```

---

### 6. Continuous Operation

The client:

* Renews certificates before expiry
* Reconnects tunnels on failure
* Runs indefinitely under systemd

---

## Certificate Lifecycle

* Certificates are short-lived.
* `max_cert_ttl` is enforced by server.
* Client renews certificate before `RENEW_BEFORE` seconds remain.
* If renewal fails, retry logic continues.

Token validity is enforced server-side.

---

## Installation

### Step 1 – Prepare Files

```
faas-client/
├─ faas_register_tunnel.sh
├─ faas_register_tunnel@.service
├─ install.sh
└─ uninstall.sh
```

---

### Step 2 – Run Installer

```bash
cd faas-client
sudo bash install.sh
```

Prompts:

* SERVICE_USER
* DOMAIN
* PRINCIPAL
* LOCAL_PORT
* TOKEN

If user does not exist, installer can create a dedicated system user.

Installer performs:

* Creates env file
* Generates SSH key
* Installs scripts
* Enables systemd service

---

## Per-User Configuration

Each user has:

```
/etc/faas_register_tunnel/<USER>.env
```

Example:

```bash
DOMAIN="bitone.in"
TOKEN="TOKEN_VALUE"
PRINCIPAL="bitresearch2006"
LOCAL_PORT=8080
RENEW_BEFORE=300
```

Permissions: `600`, owned by service user.

---

## Service Management

Start:

```bash
sudo systemctl start faas_register_tunnel@alice
```

Stop:

```bash
sudo systemctl stop faas_register_tunnel@alice
```

Enable at boot:

```bash
sudo systemctl enable faas_register_tunnel@alice
```

Status:

```bash
sudo systemctl status faas_register_tunnel@alice
```

---

## Logs & Diagnostics

Logs:

```bash
sudo journalctl -u faas_register_tunnel@alice -f
```

Certificate inspection:

```bash
sudo ssh-keygen -Lf /home/alice/.ssh/bitone_key-cert.pub
```

---

## Security Model

* Tokens are secret
* Certificates are short-lived
* One Linux user per tunnel
* Only specified port is exposed
* Reverse tunnel bound to localhost
* No direct public SSH port exposure

---

## Troubleshooting

### Service Fails to Start

```bash
sudo journalctl -u faas_register_tunnel@alice -f
```

Common causes:

* Invalid token
* Wrong principal
* Local service not running
* VM unreachable

---

## Uninstall

```bash
sudo bash uninstall.sh
```

Actions:

* Stops service
* Disables systemd unit
* Removes env file
* Optionally removes user

---

## Conclusion

This client provides:

* Secure FAAS registration
* Zero manual SSH handling
* Multi-user isolation
* Automatic recovery

Designed for large-scale WSL and Linux tunnel clients with strict operational safety.
