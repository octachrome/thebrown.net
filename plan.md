# Plan: Add screen-timer to Nginx Configuration (screens.thebrown.net)

## Overview
Extend the existing Ansible playbook to deploy a second webapp (`screens.thebrown.net`) running as a self-contained Go executable on port 8080, alongside the existing Node.js-based Treason game (`coup.thebrown.net`). Both will be fronted by Nginx using name-based virtual host routing.

## Architecture
- **Current**: Nginx receives requests for `coup.thebrown.net` and `treason.thebrown.net` → routes to Node.js upstreams (port 8081/8082)
- **New**: Nginx receives requests for `screens.thebrown.net` → routes to Go binary (port 8080)
- **Method**: Nginx name-based virtual hosting with separate `server` blocks per domain

---

## Changes Required

### 1. Nginx Configuration (roles/nginx/templates/nginx.conf.j2)

#### Add upstream for Go app:
```nginx
upstream screen-timer {
    server localhost:8080;
}
```

#### Add new server block for screens.thebrown.net:
```nginx
server {
  listen 80;
  server_name screens.thebrown.net;

  location / {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/htpasswd;
    proxy_pass http://screen-timer;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Authorization "";
  }
}
```

#### Notes:
- **Basic Auth**: Uses same htpasswd file as treason game (user: `treason`)
- **Auth Header Stripping**: `proxy_set_header Authorization "";` removes auth header before forwarding to Go app (so app doesn't see credentials)
- **Proxy Headers**: Include X-Forwarded headers so Go app can determine original client IP
- **Host Header**: Set properly so Go app knows which domain was requested

---

### 2. Create New `screen-timer` Ansible Role

**Directory structure:**
```
roles/screen-timer/
├── tasks/
│   └── main.yml
├── templates/
│   └── screen-timer.service.j2
└── files/
    └── [optional: default screen-timer binary or deployment helper]
```

#### roles/screen-timer/tasks/main.yml:
- Create `screen_timer` user and home directory
- Copy screen-timer binary from `roles/screen-timer/files/screen-timer` to `/home/screen_timer/screen-timer`
- Set binary permissions (owner: screen_timer, mode: 0755)
- Create systemd service unit for screen-timer
- Start and enable screen-timer service
- Verify service is running

#### roles/screen-timer/templates/screen-timer.service.j2:
```ini
[Unit]
Description=Screen Timer (Go)
After=network.target

[Service]
Type=simple
ExecStart=/home/screen_timer/screen-timer  # path to Go binary
User=screen_timer
Restart=always
RestartSec=10
# Optional environment variables
# Environment="PORT=8080"
# Environment="LOG_LEVEL=info"

[Install]
WantedBy=multi-user.target
```

#### Key tasks:
1. Create `screen_timer` user and group (similar to `treason` user)
2. Create `/home/screen_timer` directory with appropriate permissions
3. Copy screen-timer binary from local filesystem (`roles/screen-timer/files/screen-timer`) to server
4. Set correct ownership and permissions (755)
5. Create systemd service template and reload daemon
6. Start/enable service

#### Binary Placement:
- Place compiled screen-timer binary at: `roles/screen-timer/files/screen-timer`
- Use Ansible `copy` module to distribute to server

---

### 3. Nginx Role Enhancement

No changes needed. The htpasswd file is already created by nginx role, and the screen timer will use the same credentials as the existing auth endpoints.

---

### 4. Update Main Playbook (server.yml)

Add the new `screen-timer` role and define screen-timer variables directly in the playbook:
```yaml
- hosts: "{{host_pattern}}"
  roles:
    - common
    - network
    - letsencrypt
    - nginx
    - node
    - role: treason
      color: green
      node_port: 8081
    - role: treason
      color: blue
      node_port: 8082
    - screen-timer     # ← NEW: Add screen-timer role
    - monitor
  vars_files:
    - local_vars.yml
  vars:
    # Config vars
    treason_user: treason
    node_version: "14.15.3"
    screen_timer_user: screen_timer        # ← NEW
    screen_timer_port: 8080                # ← NEW
    # Computed vars
    node_home: "/home/{{treason_user}}/node-{{node_version}}"
    passive_color: "{{'blue' if active_color == 'green' else 'green'}}"
    ansible_python_interpreter: /usr/bin/python3
```

---

## Deployment Steps

### Initial Deployment:
1. **Compile screen-timer binary locally** (e.g., Go binary for target platform)
2. **Place binary** at `roles/screen-timer/files/screen-timer` in this repo
3. **Update server.yml** to include screen-timer role
4. **Create screen-timer role** with tasks, templates, and service definition
5. **Update nginx.conf.j2** to add screen-timer upstream and server block
6. **Run playbook**: `./run-ansible.sh`
7. **Verify** nginx config and services are running

### Updating Screen Timer Binary:
1. Compile new screen-timer binary locally
2. Replace `roles/screen-timer/files/screen-timer` in repo
3. Run playbook with tag: `./run-ansible.sh --tags screen-timer`
4. Or manually: `ansible-playbook server.yml --tags screen-timer`

---

## Testing Checklist

- [ ] Verify coup.thebrown.net still works (no auth on main location)
- [ ] Verify treason.thebrown.net still works (no auth on main location)
- [ ] Verify screens.thebrown.net requires basic auth
- [ ] Test auth with: `curl -u treason:PASSWORD http://screens.thebrown.net/`
- [ ] Test that requests to each domain route to correct upstream
- [ ] Verify screen-timer service is running: `systemctl status screen-timer`
- [ ] Check nginx config syntax: `nginx -t`
- [ ] Monitor logs during request: `tail -f /var/log/nginx/access.log`
