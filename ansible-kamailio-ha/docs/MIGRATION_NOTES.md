# Migration notes from the original kamailio-ha repository

## Imported baseline

The original `.ansible/` tree used:
- Debian Buster repository `kamailio54` (Kamailio 5.4);
- interface facts such as `ansible_eth1.ipv4.address`;
- static `copy` of full /etc configuration trees;
- service restarts inside each role;
- a Keepalived VIP model for two Kamailio/RTPengine nodes;
- local Redis configuration;
- a bundled RTPengine 8.5.1.2 .deb.

This is useful historical material, but it is not suitable as-is for the target design.

## Target changes required

1. Replace OS-specific interface facts with explicit interface/address variables and validated facts.
2. Pin supported OS and package versions after the platform decision.
3. Convert static config files into Jinja2 templates and use Ansible handlers for controlled restarts.
4. Separate package installation, configuration, validation and service activation.
5. Add idempotent config validation (Kamailio `-c`, Asterisk config checks, RTPengine config/service checks).
6. Replace the old Kamailio 5.4 dispatcher assumptions with the approved dispatcher design; algorithm 10 requires unique DUID per destination and configured hash size.
7. Keep RTPengine NG control private to the Kamailio/RTPengine network; media port range must be reflected in firewall policy.
8. Implement Patroni, DCS and HAProxy roles separately; never put PostgreSQL writer routing into SIP routing.
9. Model four Asterisk nodes as independent instances; queue/agent distribution needs a separate ACD/ownership design.
10. Exclude old inventories, public host IPs, SSH keys, local root files and bundled binary packages from the target implementation.

## Not imported as executable automation

The old `.ansible_do/` tree includes a hard-coded inventory with public cloud IP addresses and a second experimental copy of roles. It is deliberately not carried into this branch as an executable inventory.

## Validation status

No playbook in this directory has been run against any host. Syntax checks and molecule/integration tests remain to be added. Do not treat a successful syntax check as evidence of deployment correctness.
