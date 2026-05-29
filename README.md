# <img src=".gitlab/ansible.png" alt="ansible" height="30"/> install-keepalived-vip

::include{file=.gitlab/badges.md}

Ansible Role konfiguracją wirtualny adresu IP (VIP).

---

## Architektura / Flow

```mermaid
flowchart TD
    Start([Start roli]) --> Install[Install keepalived<br/>ansible.builtin.package]
    Install --> UserCheck{vrrp_scripts<br/>zdefiniowane?}
    UserCheck -->|tak| CreateUser[Utworz uzytkownika<br/>keepalived_script]
    UserCheck -->|nie| MkDir
    CreateUser --> MkDir[Utworz katalog<br/>/etc/keepalived/]
    MkDir --> ComputePeers[Compute unicast peers<br/>set_fact: _keepalived_unicast_peers]

    ComputePeers --> PeersLogic{Priorytet<br/>rozwiazania peerow}
    PeersLogic -->|1| ExplicitPeers[unicast.peers<br/>jawna lista IP]
    PeersLogic -->|2| GroupPeers[unicast.peers_group<br/>lub in_keepalived.unicast_peers_group]
    PeersLogic -->|3| PlayHosts[ansible_play_batch<br/>fallback]

    ExplicitPeers --> Validate
    GroupPeers --> ExtractIP[Extract ansible_facts.default_ipv4<br/>z hostvars, usun self]
    PlayHosts --> ExtractIP
    ExtractIP --> Validate[Validate unicast peers<br/>assert length > 0]

    Validate -->|fail| Error([FAIL: no peers found])
    Validate -->|ok| Render[Render keepalived.conf<br/>template + validate -t -f]

    Render --> Notify[Notify handler<br/>Restart keepalived]
    Notify --> ServiceMgr{ansible_facts.service_mgr}
    ServiceMgr -->|systemd| Systemd[systemd: enable + start<br/>daemon_reload]
    ServiceMgr -->|inne| Skip[Debug: skip enable/start]
    Systemd --> End([Koniec])
    Skip --> End

    style ComputePeers fill:#e1f5ff
    style Validate fill:#fff3cd
    style Error fill:#f8d7da
    style Render fill:#d4edda
```

### Handlery

```mermaid
flowchart LR
    Trigger[Notify: Restart keepalived] --> Listen{ansible_facts.service_mgr}
    Listen -->|systemd| H1[ansible.builtin.systemd<br/>state: restarted]
    Listen -->|inne| H2[ansible.builtin.service<br/>state: restarted]
```

## Wymagania

- Debian/Ubuntu lub RHEL/Alma/Rocky (EL)
- Uprawnienia `become: true`

## Najprostsze użycie

W playbooku:

```yaml
- role: install-keepalived-vip
  vars:
    in_keepalived:
      instances:
        - name: VI_HAPROXY
          state: BACKUP
          interface: eth0
          virtual_router_id: 51
          priority: 100
          virtual_ipaddresses:
            - "10.10.10.10/24 dev eth0"
```

## Unicast (bez multicastu)

```yaml
in_keepalived:
  instances:
    - name: VI_HAPROXY
      interface: eth0
      virtual_router_id: 51
      priority: 110
      unicast:
        enabled: true
        # Keepalived wymaga IP (hostname nie przejdzie walidacji).
        # Jeśli pominiesz `src_ip`/`peers` (albo ustawisz `peers: []`), template spróbuje
        # uzupełnić je na podstawie zebranych faktów (`ansible_facts.default_ipv4.address`)
        # z hostów w bieżącym batchu playu (ansible_play_batch). Możesz wskazać grupę:
        #   unicast.peers_group: haproxy
        # lub globalnie:
        #   in_keepalived.unicast_peers_group: haproxy
        peers: []
      virtual_ipaddresses:
        - "10.10.10.10/24 dev eth0"
```

Jeśli unicast jest włączony i nie uda się znaleźć żadnych peerów (brak `peers`,
brak hostów w `ansible_play_batch` / grupie oraz brak faktów), rola przerwie wykonanie
z czytelnym błędem.

---

## Zmienne

- `in_keepalived.instances` (list) – lista instancji VRRP (wymagana gdy enabled)
- `in_keepalived.instances[].extra_lines` – dodatkowe linie w `vrrp_instance` (np. `nopreempt`, `garp_master_delay 1`)

---

::include{file=.gitlab/footer.md}
