# Salt Master Restructuring — App Installs, Config Management & Onboarding

Restructure the Salt Master to support general application installations, configuration management, and automated new-machine onboarding across Ubuntu, RHEL, and Windows — **without breaking the existing Kopia backup infrastructure**.

---

## Current State Summary

What already exists on the Salt Master:

| Component | Location | Purpose |
|---|---|---|
| Kopia Windows backup | `/srv/salt/kopia/windows.sls` + `files/` | Windows backup states & scripts |
| Kopia Linux backup | `/srv/salt/kopia/linux.sls` + `files/linux/` | Linux backup states & scripts |
| Kopia pillar | `/srv/pillar/kopia/` (`init.sls`, `daily.sls`, `weekly.sls`, `linux_defaults.sls`) | Backup configuration |
| Kopia alerting | `/srv/salt/kopia/kopia_alerting.yaml`, `send_alert.py`, `/srv/salt/_runners/kopia_alert.py` | Backup alerts |
| Kopia reactor | `/etc/salt/master.d/kopia_reactor.conf`, `/srv/reactor/kopia_alert.sls` | Event-driven alerting |
| Salt minion upgrade | `/srv/salt/upgrade_minion.sls`, `/srv/pillar/salt_minion.sls` | Windows minion upgrades |

> [!IMPORTANT]
> **Zero changes to any existing Kopia or upgrade_minion files.** Everything below is purely additive.

---

## Proposed Directory Structure

```
/etc/salt/
├── master.d/
│   └── kopia_reactor.conf              # EXISTING — no changes

/srv/
├── pillar/
│   ├── top.sls                         # [MODIFY] — add app & onboarding targets
│   │
│   ├── kopia/                          # EXISTING — no changes
│   │   ├── init.sls
│   │   ├── daily.sls
│   │   ├── weekly.sls
│   │   └── linux_defaults.sls
│   │
│   ├── salt_minion.sls                 # EXISTING — no changes
│   │
│   ├── onboarding/                     # [NEW] — default apps & configs for new machines
│   │   └── init.sls
│   │
│   ├── wazuh/                          # [NEW] — Wazuh agent config
│   │   └── init.sls
│   │
│   ├── crowdstrike/                    # [NEW] — CrowdStrike config
│   │   └── init.sls
│   │
│   └── common/                         # [NEW] — shared infra config (DNS, NTP, etc.)
│       └── init.sls
│
├── reactor/                            # EXISTING — no changes
│   └── kopia_alert.sls
│
└── salt/
    ├── top.sls                         # [NEW] — master state routing file
    │
    ├── _runners/                       # EXISTING — no changes
    │   └── kopia_alert.py
    │
    ├── kopia/                          # EXISTING — no changes
    │   ├── windows.sls
    │   ├── linux.sls
    │   ├── kopia_alerting.yaml
    │   ├── send_alert.py
    │   └── files/
    │       ├── (Windows scripts)
    │       └── linux/
    │           └── (Linux scripts)
    │
    ├── upgrade_minion.sls              # EXISTING — no changes
    │
    ├── common/                         # [NEW] — shared baseline configs
    │   ├── init.sls                    #   routes by OS
    │   ├── ntp.sls
    │   ├── dns.sls
    │   └── hardening.sls
    │
    ├── onboarding/                     # [NEW] — new machine bootstrap
    │   └── init.sls                    #   orchestrates: common → apps → kopia
    │
    ├── wazuh_agent/                    # [NEW] — Wazuh agent deployment
    │   ├── init.sls                    #   auto-routes by OS grain (install)
    │   ├── ubuntu.sls
    │   ├── rhel.sls
    │   ├── windows.sls
    │   ├── uninstall.sls               # [NEW] OS router for uninstall
    │   ├── uninstall_ubuntu.sls        # [NEW]
    │   ├── uninstall_rhel.sls          # [NEW]
    │   ├── uninstall_windows.sls       # [NEW]
    │   └── files/
    │       ├── wazuh-agent-4.14.5-1.msi
    │       ├── wazuh-agent_4.14.5-1_amd64.deb
    │       └── wazuh-agent-4.14.5-1.x86_64.rpm
    │
    ├── crowdstrike/                    # [NEW] — CrowdStrike Falcon
    │   ├── init.sls
    │   ├── ubuntu.sls
    │   ├── rhel.sls
    │   ├── windows.sls
    │   ├── uninstall.sls               # [NEW] OS router for uninstall
    │   ├── uninstall_ubuntu.sls        # [NEW]
    │   ├── uninstall_rhel.sls          # [NEW]
    │   ├── uninstall_windows.sls       # [NEW]
    │   └── files/
    │       ├── CrowdStrike.msi
    │       ├── falcon-sensor_amd64.deb
    │       └── falcon-sensor.x86_64.rpm
    │
    └── <future_app>/                   # Pattern for any future app
        ├── init.sls                    #   install (OS router)
        ├── ubuntu.sls
        ├── rhel.sls
        ├── windows.sls
        ├── uninstall.sls               #   uninstall (OS router)
        ├── uninstall_ubuntu.sls
        ├── uninstall_rhel.sls
        ├── uninstall_windows.sls
        └── files/
```

---

## Design Principles

1. **`init.sls` as OS router** — Every app module's `init.sls` detects the OS via grains and includes the correct sub-state. No caller needs to know the OS.
2. **`uninstall.sls` as OS router** — Mirror of `init.sls` but routes to `uninstall_<os>.sls`. Stops the service, removes the package, and cleans up artifacts.
3. **Pillar per app** — Each app gets its own pillar namespace (`wazuh`, `crowdstrike`, `common`). No pillar key collisions.
4. **Onboarding as orchestration** — `onboarding.init` is a meta-state that includes common + all standard apps + kopia in the correct order.
5. **Files served via `salt://`** — All installers live under each app's `files/` directory for air-gapped environments.

---

## Proposed Changes

### Pillar Layer

---

#### [MODIFY] [top.sls](file:///srv/pillar/top.sls)

Add app pillars and onboarding pillar to all minions:

```yaml
# === Salt Pillar Top File ===
# Assigns pillar data based on minion grains
#
# Merge order (Salt recursive merge):
#   1. common           → shared infra config (DNS, NTP servers)
#   2. onboarding       → list of standard apps for new machines
#   3. kopia            → backup config (base)
#   4. kopia.daily/weekly → backup schedule overlay
#   5. wazuh            → Wazuh agent config
#   6. crowdstrike      → CrowdStrike config
#   7. salt_minion      → minion self-upgrade config

base:
  # ── Universal (all minions) ──
  '*':
    - common
    - onboarding

  # ── Kopia backup (by OS) ──
  'os:Windows':
    - match: grain
    - kopia
    - salt_minion

  'os:Linux':
    - match: grain
    - kopia
    - kopia.linux_defaults

  # ── Kopia schedule overlays (by role) ──
  'role:daily':
    - match: grain
    - kopia.daily

  'role:weekly':
    - match: grain
    - kopia.weekly

  # ── App pillars (all minions) ──
  # Only applied if the app is enabled in pillar
  'G@os:Windows or G@os:Ubuntu or G@os:RedHat':
    - match: compound
    - wazuh
    - crowdstrike
```

> [!NOTE]
> The compound matcher `G@os:Windows or G@os:Ubuntu or G@os:RedHat` ensures app pillars reach all three OS families. You can simplify to `'*'` if all minions should receive these pillars.

---

#### [NEW] /srv/pillar/common/init.sls

Shared infrastructure settings consumed by `common/` states:

```yaml
# === Common Infrastructure Pillar ===
common:
  ntp:
    servers:
      - 192.168.1.10
      - 192.168.1.11
  
  dns:
    nameservers:
      - 192.168.1.5
      - 192.168.1.6
    search_domains:
      - corp.local
  
  # Optional: syslog, proxy, etc.
```

---

#### [NEW] /srv/pillar/onboarding/init.sls

Defines which apps are standard for new machines:

```yaml
# === Onboarding Pillar ===
# Controls which apps are installed during new machine onboarding.
# Set enabled: false to skip an app.

onboarding:
  # Apply common baseline configs (NTP, DNS, hardening)
  apply_common: true

  # Standard apps for all new machines
  apps:
    wazuh_agent:
      enabled: true
    crowdstrike:
      enabled: true
    kopia:
      enabled: true

  # Future apps just add a key here:
  # <app_name>:
  #   enabled: true
```

---

#### [NEW] /srv/pillar/wazuh/init.sls

```yaml
# === Wazuh Agent Pillar ===
wazuh:
  manager_ip: '192.168.158.20'
  registration_key: 'YOUR_REGISTRATION_KEY'
  agent_group: 'default'
  
  version: '4.14.5'
  
  packages:
    windows: 'wazuh-agent-4.14.5-1.msi'
    ubuntu: 'wazuh-agent_4.14.5-1_amd64.deb'
    rhel: 'wazuh-agent-4.14.5-1.x86_64.rpm'
```

---

#### [NEW] /srv/pillar/crowdstrike/init.sls

```yaml
# === CrowdStrike Falcon Pillar ===
crowdstrike:
  cid: 'YOUR_CUSTOMER_ID'
  
  packages:
    windows: 'CrowdStrike.msi'
    ubuntu: 'falcon-sensor_amd64.deb'
    rhel: 'falcon-sensor.x86_64.rpm'
```

---

### State Layer

---

#### [NEW] /srv/salt/top.sls — Master State Routing

> [!WARNING]
> This file is only needed if you want `state.highstate` to automatically apply states. If you prefer explicit `state.apply <module>` calls, this file is optional but still recommended for onboarding.

```yaml
# === Salt State Top File ===
# Maps minions to states. Used by: salt '*' state.highstate
#
# For targeted deployment, use: salt '<minion>' state.apply <module>

base:
  # ── Common baseline (all minions) ──
  '*':
    - common

  # ── Kopia backup (by OS) ──
  'os:Windows':
    - match: grain
    - kopia.windows

  'os:Linux':
    - match: grain
    - kopia.linux

  # ── Apps can be added here for automatic deployment ──
  # Uncomment when ready for automatic highstate deployment:
  # '*':
  #   - wazuh_agent
  #   - crowdstrike
```

> [!TIP]
> Keep app states commented in `top.sls` initially. Deploy via explicit `state.apply` until you're confident, then uncomment for highstate.

---

#### [NEW] /srv/salt/common/init.sls — Shared Baseline

```yaml
# === Common Baseline Configuration ===
# Applied to ALL minions regardless of OS.
# Individual sub-states handle OS-specific implementation.

include:
  - common.ntp
  - common.dns
  # - common.hardening    # Uncomment when ready
```

---

#### [NEW] /srv/salt/common/ntp.sls

```yaml
# === NTP Configuration ===
{%- set ntp = salt['pillar.get']('common:ntp', {}) %}
{%- set servers = ntp.get('servers', []) %}

{%- if grains['os_family'] == 'Windows' %}

win_ntp_configure:
  cmd.run:
    - name: >
        w32tm /config
        /manualpeerlist:"{{ servers | join(' ') }}"
        /syncfromflags:manual /reliable:yes /update
    - shell: cmd

win_ntp_restart:
  service.running:
    - name: W32Time
    - enable: True
    - watch:
      - cmd: win_ntp_configure

{%- elif grains['os_family'] == 'Debian' %}

chrony_pkg:
  pkg.installed:
    - name: chrony

chrony_conf:
  file.managed:
    - name: /etc/chrony/chrony.conf
    - contents: |
        # Managed by SaltStack — DO NOT EDIT
        {%- for server in servers %}
        server {{ server }} iburst
        {%- endfor %}
        driftfile /var/lib/chrony/chrony.drift
        makestep 1.0 3
        rtcsync
    - require:
      - pkg: chrony_pkg

chrony_service:
  service.running:
    - name: chrony
    - enable: True
    - watch:
      - file: chrony_conf

{%- elif grains['os_family'] == 'RedHat' %}

chrony_pkg:
  pkg.installed:
    - name: chrony

chrony_conf:
  file.managed:
    - name: /etc/chrony.conf
    - contents: |
        # Managed by SaltStack — DO NOT EDIT
        {%- for server in servers %}
        server {{ server }} iburst
        {%- endfor %}
        driftfile /var/lib/chrony/drift
        makestep 1.0 3
        rtcsync
    - require:
      - pkg: chrony_pkg

chrony_service:
  service.running:
    - name: chronyd
    - enable: True
    - watch:
      - file: chrony_conf

{%- endif %}
```

---

#### [NEW] /srv/salt/common/dns.sls

```yaml
# === DNS Configuration ===
{%- set dns = salt['pillar.get']('common:dns', {}) %}
{%- set nameservers = dns.get('nameservers', []) %}
{%- set search = dns.get('search_domains', []) %}

{%- if grains['os_family'] in ('Debian', 'RedHat') %}

# For Linux: manage /etc/resolv.conf
# NOTE: On systemd-resolved systems, consider managing resolved.conf instead
dns_resolv_conf:
  file.managed:
    - name: /etc/resolv.conf
    - contents: |
        # Managed by SaltStack — DO NOT EDIT
        {%- for ns in nameservers %}
        nameserver {{ ns }}
        {%- endfor %}
        {%- if search %}
        search {{ search | join(' ') }}
        {%- endif %}

{%- elif grains['os_family'] == 'Windows' %}

# Windows DNS is typically managed via DHCP or GPO
# Only set if explicitly configured
{%- if nameservers %}
win_dns_configure:
  cmd.run:
    - name: >
        $adapters = Get-NetAdapter | Where-Object { $_.Status -eq 'Up' };
        foreach ($a in $adapters) {
          Set-DnsClientServerAddress -InterfaceIndex $a.ifIndex
          -ServerAddresses {{ nameservers | join(',') }}
        }
    - shell: powershell
{%- endif %}

{%- endif %}
```

---

#### [NEW] /srv/salt/wazuh_agent/init.sls — OS Router

This is the **standard pattern for all app modules**:

```yaml
# === Wazuh Agent Deployment ===
# Auto-routes to the correct OS-specific state based on grains.

{%- if grains['os'] == 'Ubuntu' %}
include:
  - wazuh_agent.ubuntu

{%- elif grains['os'] == 'RedHat' or grains['os'] == 'CentOS' or grains['os'] == 'Rocky' or grains['os'] == 'AlmaLinux' %}
include:
  - wazuh_agent.rhel

{%- elif grains['os_family'] == 'Windows' %}
include:
  - wazuh_agent.windows

{%- else %}
wazuh_unsupported_os:
  test.fail_without_changes:
    - name: "Wazuh agent: unsupported OS '{{ grains['os'] }}'"
{%- endif %}
```

---

#### [NEW] /srv/salt/wazuh_agent/ubuntu.sls

```yaml
# === Wazuh Agent — Ubuntu ===
{%- set wazuh = salt['pillar.get']('wazuh', {}) %}
{%- set pkg = wazuh.get('packages', {}).get('ubuntu', '') %}

wazuh_agent_install_deb:
  file.managed:
    - name: /tmp/{{ pkg }}
    - source: salt://wazuh_agent/files/{{ pkg }}
    - skip_verify: True

  cmd.run:
    - name: dpkg -i /tmp/{{ pkg }}
    - unless: dpkg -l wazuh-agent 2>/dev/null | grep -q "^ii"
    - require:
      - file: wazuh_agent_install_deb

wazuh_agent_config:
  file.managed:
    - name: /var/ossec/etc/ossec.conf
    - source: salt://wazuh_agent/files/ossec.conf.jinja
    - template: jinja
    - require:
      - cmd: wazuh_agent_install_deb

wazuh_agent_service:
  service.running:
    - name: wazuh-agent
    - enable: True
    - watch:
      - file: wazuh_agent_config
```

---

#### [NEW] /srv/salt/wazuh_agent/rhel.sls

```yaml
# === Wazuh Agent — RHEL/Rocky/Alma ===
{%- set wazuh = salt['pillar.get']('wazuh', {}) %}
{%- set pkg = wazuh.get('packages', {}).get('rhel', '') %}

wazuh_agent_install_rpm:
  file.managed:
    - name: /tmp/{{ pkg }}
    - source: salt://wazuh_agent/files/{{ pkg }}
    - skip_verify: True

  cmd.run:
    - name: rpm -ivh /tmp/{{ pkg }}
    - unless: rpm -q wazuh-agent
    - require:
      - file: wazuh_agent_install_rpm

wazuh_agent_config:
  file.managed:
    - name: /var/ossec/etc/ossec.conf
    - source: salt://wazuh_agent/files/ossec.conf.jinja
    - template: jinja
    - require:
      - cmd: wazuh_agent_install_rpm

wazuh_agent_service:
  service.running:
    - name: wazuh-agent
    - enable: True
    - watch:
      - file: wazuh_agent_config
```

---

#### [NEW] /srv/salt/wazuh_agent/windows.sls

```yaml
# === Wazuh Agent — Windows ===
{%- set wazuh = salt['pillar.get']('wazuh', {}) %}
{%- set pkg = wazuh.get('packages', {}).get('windows', '') %}

wazuh_agent_stage:
  file.managed:
    - name: 'C:\Windows\Temp\{{ pkg }}'
    - source: salt://wazuh_agent/files/{{ pkg }}
    - skip_verify: True

wazuh_agent_install_msi:
  cmd.run:
    - name: >
        msiexec /i "C:\Windows\Temp\{{ pkg }}" /qn
        WAZUH_MANAGER="{{ wazuh.get('manager_ip', '') }}"
        WAZUH_REGISTRATION_KEY="{{ wazuh.get('registration_key', '') }}"
        WAZUH_AGENT_GROUP="{{ wazuh.get('agent_group', 'default') }}"
    - shell: cmd
    - unless: 'sc query WazuhSvc'
    - require:
      - file: wazuh_agent_stage

wazuh_agent_service:
  service.running:
    - name: WazuhSvc
    - enable: True
    - require:
      - cmd: wazuh_agent_install_msi
```

---

#### [NEW] /srv/salt/crowdstrike/init.sls — Same Pattern

```yaml
# === CrowdStrike Falcon Deployment ===
{%- if grains['os'] == 'Ubuntu' %}
include:
  - crowdstrike.ubuntu
{%- elif grains['os_family'] == 'RedHat' %}
include:
  - crowdstrike.rhel
{%- elif grains['os_family'] == 'Windows' %}
include:
  - crowdstrike.windows
{%- else %}
crowdstrike_unsupported:
  test.fail_without_changes:
    - name: "CrowdStrike: unsupported OS '{{ grains['os'] }}'"
{%- endif %}
```

*(OS-specific install `.sls` files follow the same pattern as Wazuh — `file.managed` + `cmd.run` + `service.running`)*

---

### Uninstall States

Every app module gets an `uninstall.sls` OS router + per-OS uninstall states. The pattern is: **stop service → remove package → clean up files**.

#### Usage

```bash
# Uninstall Wazuh from a specific minion
salt 'server-01' state.apply wazuh_agent.uninstall

# Uninstall CrowdStrike from all Ubuntu machines
salt -G 'os:Ubuntu' state.apply crowdstrike.uninstall

# Uninstall any app — same pattern
salt '<minion>' state.apply <app_name>.uninstall
```

---

#### [NEW] /srv/salt/wazuh_agent/uninstall.sls — Uninstall OS Router

```yaml
# === Wazuh Agent Uninstall ===
# Auto-routes to the correct OS-specific uninstall state.
#
# Usage: salt '<minion>' state.apply wazuh_agent.uninstall

{%- if grains['os'] == 'Ubuntu' %}
include:
  - wazuh_agent.uninstall_ubuntu

{%- elif grains['os'] == 'RedHat' or grains['os'] == 'CentOS' or grains['os'] == 'Rocky' or grains['os'] == 'AlmaLinux' %}
include:
  - wazuh_agent.uninstall_rhel

{%- elif grains['os_family'] == 'Windows' %}
include:
  - wazuh_agent.uninstall_windows

{%- else %}
wazuh_uninstall_unsupported:
  test.fail_without_changes:
    - name: "Wazuh uninstall: unsupported OS '{{ grains['os'] }}'"
{%- endif %}
```

---

#### [NEW] /srv/salt/wazuh_agent/uninstall_ubuntu.sls

```yaml
# === Wazuh Agent Uninstall — Ubuntu ===

wazuh_service_stop:
  service.dead:
    - name: wazuh-agent
    - enable: False

wazuh_agent_remove_deb:
  cmd.run:
    - name: dpkg --purge wazuh-agent
    - onlyif: dpkg -l wazuh-agent 2>/dev/null | grep -q "^ii"
    - require:
      - service: wazuh_service_stop

wazuh_cleanup_dir:
  file.absent:
    - name: /var/ossec
    - require:
      - cmd: wazuh_agent_remove_deb
```

---

#### [NEW] /srv/salt/wazuh_agent/uninstall_rhel.sls

```yaml
# === Wazuh Agent Uninstall — RHEL/Rocky/Alma ===

wazuh_service_stop:
  service.dead:
    - name: wazuh-agent
    - enable: False

wazuh_agent_remove_rpm:
  cmd.run:
    - name: rpm -e wazuh-agent
    - onlyif: rpm -q wazuh-agent
    - require:
      - service: wazuh_service_stop

wazuh_cleanup_dir:
  file.absent:
    - name: /var/ossec
    - require:
      - cmd: wazuh_agent_remove_rpm
```

---

#### [NEW] /srv/salt/wazuh_agent/uninstall_windows.sls

```yaml
# === Wazuh Agent Uninstall — Windows ===
{%- set wazuh = salt['pillar.get']('wazuh', {}) %}
{%- set pkg = wazuh.get('packages', {}).get('windows', '') %}

wazuh_service_stop_win:
  service.dead:
    - name: WazuhSvc
    - enable: False

wazuh_agent_uninstall_msi:
  cmd.run:
    - name: >
        msiexec /x "C:\Windows\Temp\{{ pkg }}" /qn /norestart
    - onlyif: 'sc query WazuhSvc'
    - require:
      - service: wazuh_service_stop_win

wazuh_cleanup_dir_win:
  file.absent:
    - name: 'C:\Program Files (x86)\ossec-agent'
    - require:
      - cmd: wazuh_agent_uninstall_msi
```

---

#### [NEW] /srv/salt/crowdstrike/uninstall.sls — Same Pattern

```yaml
# === CrowdStrike Falcon Uninstall ===
# Usage: salt '<minion>' state.apply crowdstrike.uninstall

{%- if grains['os'] == 'Ubuntu' %}
include:
  - crowdstrike.uninstall_ubuntu
{%- elif grains['os_family'] == 'RedHat' %}
include:
  - crowdstrike.uninstall_rhel
{%- elif grains['os_family'] == 'Windows' %}
include:
  - crowdstrike.uninstall_windows
{%- else %}
crowdstrike_uninstall_unsupported:
  test.fail_without_changes:
    - name: "CrowdStrike uninstall: unsupported OS '{{ grains['os'] }}'"
{%- endif %}
```

*(OS-specific uninstall `.sls` files follow the same pattern — `service.dead` → package removal → `file.absent` cleanup)*

> [!NOTE]
> **Uninstall is always independent** — it doesn't check the onboarding pillar. You can uninstall any app at any time on any minion regardless of how it was installed.

---

#### [NEW] /srv/salt/onboarding/init.sls — New Machine Bootstrap

This is the **meta-state** you run on newly onboarded machines. It applies everything in the right order:

```yaml
# === New Machine Onboarding ===
# Applies the full standard stack to a freshly onboarded minion.
#
# Usage:
#   salt '<new-minion>' state.apply onboarding
#
# What it does (in order):
#   1. Common baseline (NTP, DNS, hardening)
#   2. Security agents (Wazuh, CrowdStrike)
#   3. Backup agent (Kopia — auto-selects Windows/Linux)

{%- set onboard = salt['pillar.get']('onboarding', {}) %}

# ── Step 1: Common baseline ──
{%- if onboard.get('apply_common', true) %}
include:
  - common
{%- endif %}

# ── Step 2: Standard apps ──
{%- set apps = onboard.get('apps', {}) %}

{%- if apps.get('wazuh_agent', {}).get('enabled', false) %}
onboard_wazuh:
  module.run:
    - name: state.apply
    - mods: wazuh_agent
    - require:
      - sls: common
{%- endif %}

{%- if apps.get('crowdstrike', {}).get('enabled', false) %}
onboard_crowdstrike:
  module.run:
    - name: state.apply
    - mods: crowdstrike
    - require:
      - sls: common
{%- endif %}

# ── Step 3: Backup (auto-selects OS) ──
{%- if apps.get('kopia', {}).get('enabled', false) %}
{%- if grains['os_family'] == 'Windows' %}
onboard_kopia:
  module.run:
    - name: state.apply
    - mods: kopia.windows
{%- else %}
onboard_kopia:
  module.run:
    - name: state.apply
    - mods: kopia.linux
{%- endif %}
{%- endif %}
```

> [!TIP]
> **Usage**: After accepting a new minion's key, run one command:
> ```bash
> salt '<new-minion>' state.apply onboarding
> ```
> This installs and configures everything: NTP, DNS, Wazuh, CrowdStrike, and Kopia — in the correct order.

---

## User Review Required

> [!IMPORTANT]
> **State top.sls**: Currently you don't have `/srv/salt/top.sls`. Creating it enables `state.highstate`. If you prefer to keep using explicit `state.apply` only, we can skip this file and rely solely on targeted commands.

> [!IMPORTANT]
> **Pillar scope for apps**: The plan sends `wazuh` and `crowdstrike` pillar to all three OS families via compound matcher. If some machines should NOT get these apps, we can use a grain-based opt-in (e.g., `role:managed`) instead of `'*'`.

> [!WARNING]
> **Onboarding state uses `module.run`**: This calls `state.apply` from within a state for ordered execution. An alternative is a Salt **orchestrator** (`salt-run state.orch`). Let me know if you prefer orchestration.

---

## Open Questions

> [!IMPORTANT]
> **1. Which apps are standard for onboarding?** The plan assumes Wazuh + CrowdStrike + Kopia. Are there others (monitoring agents, log forwarders, etc.)?

> [!IMPORTANT]
> **2. Common configs scope**: What baseline configs should `common/` manage? The plan includes NTP and DNS as examples. Should it also cover:
> - SSH hardening (Linux)
> - Firewall rules
> - Local admin accounts
> - Syslog forwarding
> - Proxy settings

> [!IMPORTANT]
> **3. `state.highstate` vs explicit `state.apply`**: Do you want a `top.sls` that auto-applies states during highstate, or do you prefer purely manual `state.apply <module>` targeting?

> [!IMPORTANT]
> **4. Adding a new app in the future**: With this structure, adding a new app is:
> 1. Create `/srv/salt/<app_name>/init.sls` (OS router) + OS-specific `.sls` files
> 2. Create `/srv/pillar/<app_name>/init.sls` (config)
> 3. Add to pillar `top.sls`
> 4. Add to `onboarding/init.sls` pillar if it's standard
>
> Does this workflow make sense for your team?

---

## Verification Plan

### Automated Tests

```bash
# 1. Verify pillar data reaches all minions
salt '*' pillar.get onboarding
salt '*' pillar.get common
salt -G 'os:Windows' pillar.get wazuh
salt -G 'os:Linux' pillar.get wazuh

# 2. Dry-run common baseline
salt '<test-minion>' state.apply common test=True

# 3. Dry-run individual app install
salt '<test-minion>' state.apply wazuh_agent test=True
salt '<test-minion>' state.apply crowdstrike test=True

# 4. Dry-run individual app uninstall
salt '<test-minion>' state.apply wazuh_agent.uninstall test=True
salt '<test-minion>' state.apply crowdstrike.uninstall test=True

# 5. Dry-run full onboarding
salt '<test-minion>' state.apply onboarding test=True

# 6. Verify Kopia still works (regression test)
salt -G 'os:Windows' state.apply kopia.windows test=True
salt -G 'os:Linux' state.apply kopia.linux test=True

# 7. Check for state ID conflicts
salt '<test-minion>' state.show_sls onboarding
```

### Manual Verification

1. Onboard a test minion (one per OS) and run `state.apply onboarding`
2. Verify all services are running: `wazuh-agent`, `falcon-sensor`, Kopia timers
3. Confirm existing Kopia backups on production machines are unaffected
