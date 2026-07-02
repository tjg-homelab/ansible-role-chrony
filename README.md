# Ansible Role: chrony

[![CI](https://github.com/tjg-homelab/ansible-role-chrony/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-chrony/actions/workflows/ci.yml)

Installs and configures [chrony](https://chrony-project.org/) — a modern NTP
implementation — on Debian and RedHat family systems.

Beyond a stock time client, this role supports the full range of chrony
topologies through variables: pulling from LAN time servers, peering between
local servers, public pool fallback, serving time to a network, and driving a
local **GPS/PPS reference clock** to run your own stratum-1 server.

## Requirements

- Debian 12/13, Ubuntu 22.04/24.04, or Enterprise Linux 9
- For a GPS refclock: a configured PPS/SHM source (e.g. `gpsd`) — providing the
  reference clock daemon is outside this role's scope

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `chrony_config_file` | `/etc/chrony/chrony.conf` | Main config path |
| `chrony_source_folder` | `/etc/chrony/sources.d` | Directory for source files |
| `chrony_sources` | pool.ntp.org (pool, iburst) | List of time sources (see below) |
| `chrony_net_access` | commented | Serve time to a network, e.g. `allow 192.0.2.0/24` |
| `chrony_gps_config` | commented | Reference-clock line, e.g. `refclock SHM 0 refid GPS offset 0.200 trust prefer` |
| `chrony_leapsec_config` | commented | Leap-second source, e.g. `leapseclist /usr/share/zoneinfo/leap-seconds.list` |
| `chrony_service_state` | `started` | Service runtime state |
| `chrony_service_enabled` | `true` | Enable service at boot |

### Defining time sources

Each entry in `chrony_sources` becomes a line in a file under
`chrony_source_folder`. Group related sources by giving them the same
`location` (filename):

```yaml
chrony_sources:
  - location: local.sources     # -> /etc/chrony/sources.d/local.sources
    type: server                # server | pool | peer
    host: 192.0.2.10
    parameters: trust iburst    # any chrony source options
  - location: pool.sources
    type: pool
    host: pool.ntp.org
    parameters: iburst
```

## Example Playbooks

**Simple client (LAN server with pool fallback):**

```yaml
- hosts: servers
  roles:
    - role: chrony
      vars:
        chrony_sources:
          - location: local.sources
            type: server
            host: 192.0.2.10
            parameters: trust iburst
          - location: pool.sources
            type: pool
            host: pool.ntp.org
            parameters: iburst
```

**Stratum-1 GPS time server serving the LAN:**

```yaml
- hosts: gps_ntp
  roles:
    - role: chrony
      vars:
        chrony_net_access: "allow 192.0.2.0/24"
        chrony_gps_config: "refclock SHM 0 refid GPS offset 0.200 trust prefer"
        chrony_leapsec_config: "leapseclist /usr/share/zoneinfo/leap-seconds.list"
        chrony_sources:
          # a peer server plus public pool as sanity-check fallback
          - location: local.sources
            type: peer
            host: 192.0.2.11
            parameters: iburst
          - location: pool.sources
            type: pool
            host: pool.ntp.org
            parameters: iburst
```

Installing via `requirements.yml`:

```yaml
roles:
  - name: chrony
    src: https://github.com/tjg-homelab/ansible-role-chrony.git
    version: v1.0.0
```

## Testing

Molecule (Docker driver) converges, checks idempotence, and verifies against
Debian 12, Debian 13, and Ubuntu 24.04 containers.

```bash
pip install ansible-core molecule molecule-plugins[docker] docker
ansible-galaxy collection install community.docker ansible.posix
molecule test
```

## License

MIT

## Author

Rodney Nissen ([The Jira Guy](https://thejiraguy.com))
