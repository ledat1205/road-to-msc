# 1) TCP + SSH protocol handshake (wire-level sequence)

1. **TCP handshake**
    
    - Client → Server: `SYN` to server_ip:22
        
    - Server → Client: `SYN/ACK`
        
    - Client → Server: `ACK`
        
    - TCP connection established (possible TCP-level middlebox interference, e.g., NAT timeouts, FW).
        
2. **SSH identification exchange**
    
    - Client sends ASCII: `SSH-2.0-<client-software>` (e.g. `SSH-2.0-OpenSSH_9.0`)
        
    - Server responds: `SSH-2.0-<server-software>`
        
    - These strings are _not_ authenticated — they only advertise versions.
        
3. **KEXINIT (algorithm negotiation)**
    
    - Both sides send `SSH_MSG_KEXINIT` listing supported algorithms (KEX, host key, encryption, MAC, compression, languages).
        
    - Negotiation chooses mutually supported algorithms (server preference normally wins in OpenSSH).
        
4. **Key Exchange (KEX)** — builds the shared secret `K` and session identifier `session_id`
    
    - Typical KEX: `ecdh-sha2-nistp256` or `curve25519-sha256` (modern `curve25519` is common).
        
    - Diffie-Hellman-derived shared secret `K`.
        
    - Both compute exchange hash `H` (hash over many fields including ordered KEX payloads and the host key).
        
    - Server sends its **host key** and a **signature of H** using the server’s host private key. Client verifies signature using known host key (from `~/.ssh/known_hosts`). This prevents active MITM once host key is known/trusted.
        
    - From `K` and `H`, keys are derived via HKDF-like expansion for:
        
        - client->server encryption key
            
        - server->client encryption key
            
        - client->server MAC key
            
        - server->client MAC key
            
        - IVs etc.
            
    - **Session is now encrypted.**
        
5. **(Optional) Re-keying** after a byte threshold or time limit (`RekeyLimit`), repeats KEX to provide forward secrecy periodically.
    

# 2) Authentication phase (after KEX & encryption)

OpenSSH supports several authentication methods; server advertises what it accepts.

- **publickey** (recommended)
    
    - Client offers a key: sends public key algorithm and public key, server checks `authorized_keys` or other hooks.
        
    - If accepted, server sends a **challenge**: client signs data including `session_id` and `SSH_MSG_USERAUTH_REQUEST` payload; server verifies signature with the public key.
        
    - Important: the signature covers the session identifier, preventing replay attacks.
        
- **password**
    
    - Client sends password over encrypted channel. Server may use PAM, check `/etc/shadow`, LDAP, etc.
        
- **keyboard-interactive**
    
    - Generic prompt mechanism used by PAM/2FA. Server can request multiple prompts (OTP, password, etc).
        
- **hostbased** (rare)
    
    - Trusts client host's identity + user mapping — needs proper host keys & `hosts.equiv` style setup.
        
- **certificates** (OpenSSH CA)
    
    - User or host keys signed by a CA (`ssh-keygen -s`), server trusts the CA public key — scales better than managing `authorized_keys`.
        

# 3) SSH channels & subsystems (multiplexed over single connection)

Once authenticated, SSH multiplexes logical channels over the encrypted TCP stream. Each channel has a type and independent flow control.

Common channel types:

- `session` — interactive shell, exec commands, subsystems
    
- `direct-tcpip` — client-initiated port forwarding (local forward)
    
- `forwarded-tcpip` — server-initiated forwarded connection (remote forward)
    
- `auth-agent@openssh.com` — agent forwarding channel
    
- `x11` — X11 forwarding
    

Subsystems:

- `sftp` (subsystem) — SFTP runs over a `session` channel using its own protocol.
    
- `scp` uses `exec` to run `scp` on remote end — different semantics.
    

Port forwarding types:

- **Local (`-L`)**: `ssh -L [local_ip:]local_port:remote_host:remote_port user@jump`  
    Client listens locally; incoming conn forwarded over SSH to `remote_host:remote_port`.
    
- **Remote (`-R`)**: `ssh -R [remote_ip:]remote_port:local_host:local_port user@jump`  
    Server listens and forwards back to client side (useful for reverse SSH/tunneling).
    
- **Dynamic (`-D`)**: `ssh -D local_port user@jump`  
    Creates a SOCKS5 proxy — client acts as SOCKS proxy, SSH forwards arbitrary TCP via server.
    

# 4) Crypto primitives & algorithms (what to pick)

- **KEX**: `curve25519-sha256`, `ecdh-sha2-nistp256` (curve25519 preferred).
    
- **Host key algorithms**: `ssh-ed25519`, `ecdsa-sha2-nistp256`, `rsa-sha2-512/256`. Avoid legacy `ssh-rsa` (sha1).
    
- **Ciphers** (encryption): prefer AEAD or stream ciphers with auth: `chacha20-poly1305@openssh.com`, `aes128-ctr`, `aes256-ctr`. Avoid CBC modes.
    
- **MACs**: use `umac-64-etm@openssh.com`, `hmac-sha2-256-etm@openssh.com` (ETM = encrypt-then-MAC).
    
- **Rekeying** policy**:** `RekeyLimit` (bytes, time).
    

# 5) Practical client-side setup (commands)

- **Generate keys**
    

`# modern recommended ssh-keygen -t ed25519 -C "you@example.com" # or ssh-keygen -t rsa -b 4096 -o -a 100`

- **Copy public key to server**
    

`ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server # or append manually: cat ~/.ssh/id_ed25519.pub | ssh user@server 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'`

- **Use ssh-agent (avoid typing passphrase repeatedly)**
    

`eval "$(ssh-agent -s)" ssh-add ~/.ssh/id_ed25519   # enter passphrase once`

- **SSH config for convenience (`~/.ssh/config`)**
    

`Host myserver   HostName server.example.com   User ubuntu   IdentityFile ~/.ssh/id_ed25519   Port 22   ForwardAgent no   ServerAliveInterval 60   ServerAliveCountMax 3`

# 6) Server configuration highlights (`/etc/ssh/sshd_config`)

Key security options and meanings:

`# Basic Port 22 PermitRootLogin no                # disable direct root login PubkeyAuthentication yes PasswordAuthentication no         # force key-based auth PermitEmptyPasswords no MaxAuthTries 3 LoginGraceTime 60  # Hardening UseDNS no                         # speed up login; prevents DNS delays/misleading hostnames AllowUsers alice bob              # restrict allowed users PermitTunnel no AllowTcpForwarding no             # disable port forwarding if not needed GatewayPorts no X11Forwarding no  # Keys & algorithms (example hardened) HostKey /etc/ssh/ssh_host_ed25519_key KexAlgorithms curve25519-sha256 Ciphers chacha20-poly1305@openssh.com,aes256-ctr,aes128-ctr MACs umac-64-etm@openssh.com,hmac-sha2-256-etm@openssh.com  # Limits / logging LogLevel VERBOSE RekeyLimit 1G 1h`

- After edits: test with `sshd -t` and restart service (systemd: `systemctl restart sshd`).
    

# 7) Debugging & diagnostics

- Client verbose: `ssh -vvv user@host` — shows KEX, host key fingerprint, auth attempts, channels.
    
- Check server port/listening: `ss -tnlp | grep sshd` or `netstat -tnlp`.
    
- Server logs: `sudo journalctl -u sshd` or `/var/log/auth.log` (Debian/Ubuntu), `/var/log/secure` (RHEL/CentOS).
    
- Verify effective `sshd` config: `sshd -T` (prints normalized config).
    
- Test connection path: `telnet host 22` or `nc -vz host 22` (simple TCP reachability).
    

# 8) Advanced features & operational notes

- **Agent forwarding (`-A`)**: forwards agent socket to remote host; convenient for git/ssh hops but dangerous on untrusted hosts. Prefer `ProxyJump` instead of chaining `ssh` hops.
    
- **ControlMaster/ControlPath (connection multiplexing)**:
    

`# in ~/.ssh/config Host *   ControlMaster auto   ControlPath ~/.ssh/cm-%r@%h:%p   ControlPersist 10m`

Reuses single TCP connection for multiple sessions — faster, fewer auths.

- **ProxyJump (SSH hop)**:
    

`ssh -J jump.example.com target.example.com # in config: Host target   ProxyJump jump.example.com`

- **AuthorizedKeysCommand / AuthorizedKeysFile**: server can fetch keys from external source (LDAP, DB) dynamically.
    
- **Certificates**: use OpenSSH CA to sign user or host keys; server trusts CA pubkey (`TrustedUserCAKeys`) rather than many authorized_keys.
    
- **Two-factor / PAM**: enable `ChallengeResponseAuthentication yes` and configure PAM modules (e.g., Google Authenticator). Beware of interactions with `PasswordAuthentication` and `UsePAM`.
    

# 9) Connection lifecycle internals (what the server runs)

- `sshd` process (initial) accepts TCP → forks/threads a child for the connection.
    
- Child handles KEX, auth; on successful auth, a session child spawns a shell or subsystem (`/bin/bash`, `sftp-server`, etc).
    
- `sshd` may record login events to auth logs and trigger PAM.
    

# 10) Security hardening checklist (practical)

- Use key-based auth; disable password auth (`PasswordAuthentication no`).
    
- Disable root login (`PermitRootLogin no`) and use `AllowUsers`/`AllowGroups`.
    
- Use modern algorithms: `ed25519` host & user keys, `curve25519` KEX, AEAD ciphers.
    
- Limit `MaxAuthTries`, `LoginGraceTime`, enable `Fail2Ban` or rate-limiting.
    
- Disable agent forwarding unless needed; restrict port forwarding.
    
- Use host/user certificates for scale.
    
- Keep OpenSSH up to date for CVE fixes.
    
- Consider `sshd` chroot for SFTP-only users: `Match` + `ChrootDirectory`.
    

# 11) Example end-to-end flow (publickey auth: precise data flow)

1. TCP established.
    
2. Identification exchange.
    
3. KEXINIT from client & server → choose `curve25519-sha256`.
    
4. Perform DH exchange → both compute `K`, `H`.
    
5. Server sends host public key + signature over `H`.
    
6. Client verifies host key against `known_hosts`. If unknown, prompt to accept fingerprint.
    
7. Symmetric keys derived. Channel encrypted.
    
8. Client requests `userauth` with method `publickey` and includes public key blob.
    
9. Server checks `authorized_keys` and replies `PK_OK` (or rejects).
    
10. Client signs the `session identifier` + `userauth request` and sends signature to server.
    
11. Server verifies signature using the public key from `authorized_keys`.
    
12. If OK → server grants user session, server opens `session` channel and executes shell or subsystem.
    

# 12) Useful commands / snippets (cheat sheet)

`# Generate keys ssh-keygen -t ed25519 -C "me"  # Copy key to server ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host  # Test with verbosity ssh -vvv user@host  # Dump server effective configuration sudo sshd -T  # Test sshd config syntax sudo sshd -t  # Check listening sshd ss -tnlp | grep sshd  # Watch auth logs sudo tail -F /var/log/auth.log   # Debian/Ubuntu sudo tail -F /var/log/secure     # RHEL/CentOS  # Create SOCKS proxy ssh -D 1080 user@jump  # Local port forward ssh -L 8080:internal.example:80 user@jump  # Remote port forward (remote listens) ssh -R 2222:localhost:22 user@jump`