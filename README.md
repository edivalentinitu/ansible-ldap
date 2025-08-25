# ansible-ldap

# Deployment Playbooks for OpenLDAP

This is original taken from [deployments](https://github.com/cyverse-de/deployments).

The playbooks in this directory can be used to install OpenLDAP for the Discovery Environment (DE). This step can be
skipped if an existing LDAP service will be used. Note: as of this writing, the DE has only been used with an OpenLDAP
instance using an RFC 2307 schema.

## Preq
* We need to install **community.general** see [Issue](https://github.com/weareinteractive/ansible-ufw/issues/26)
  ```
  ansible-galaxy collection install community.general
  ```

## Playbooks

### slapd.yml

This playbook installs OpenLDAP itself. This is the only playbook required for this intallation step. After installing
OpenLDAP itself, this playbook installs the RFC 2307 schema and defines the following entities:

| Entity      | Entity Type | Description                                                                        |
| ------      | ----------- | -----------                                                                        |
| Base Domain | Domain      | This is the base for all of the other domains, and the name is configurable.       |
| People      | Domain      | This is the base entity for all DE users.                                          |
| Groups      | Domain      | This is the base entity for some groups of users.                                  |
| everyone    | Group       | This is the group containing all DE users.                                         |
| de_admins   | Group       | Members of this group will have administrative privileges in the DE.               |
| de_grouper  | User        | This is an account used by DE services to access Grouper.                          |
| ldap_reader | User        | This is the account used by Grouper and Keycloak to access LDAP.                   |
| ldap_portal | User        | This is the account used by user-portal to access LDAP. Member of de_admins group. |
| anonymous   | User        | This is the account for Anonymous-Public Access. Member of everyone group. |



## Inventory Setup

The inventory for this set of playbooks contains only the LDAP node itself:

```
[ldap]
ldap.example.org
```

## Group Variable Setup

This set of playbooks requires a few group variables:

| Variable             | Description                                                |
| --------             | -----------                                                |
| dn_suffix            | This is the name of the base domain in the LDAP directory. |
| root_password        | This is the password of the LDAP administrative account.   |
| de_grouper_password  | This is the password of the `de_grouper` account.          |
| ldap_reader_password | This is the password of the `ldap_reader` account.         |
| ldap_portal_password | This is the password of the `ldap_portal` account.         |

Example group variables file:

``` yaml
ldap:
  dn_suffix: "dc=example,dc=org"
  root_password: "some-password"
  de_grouper_password: "some-other-password"
  ldap_reader_password: "yet-another-password"
```

## LDAP Admin Account

The name of the administrative account is determined by the domain name (`ldap.dn_suffix`). For example, if the domain
name is `dc=example,dc=org`, then the manager account is `cn=Manager,dc=example,dc=org`.

## Installing LDAP

```yaml
# vi slapd.yaml
---
- name: Install and Configure LDAP
  hosts: ldap
  roles:
    - role: de-slapd
      vars:
        dn_suffix: "dc=tugraz,dc=at"
        root_password: "notprod"
        de_grouper_password: "notprod"
        ldap_reader_password: "notprod"
```

# Run Playbook
This installation invoves merely running the playbook:

```bash
ansible-playbook -i inventory/ slapd.yml --user root
```

# Local development (Ubuntu/Debian)
To test/develop the LDAP server locally, make sure to:
1. Change the inventory/ldap file to have **localhost** or **127.0.0.1**
2. The previous command should be: (**-K** prompts the user for the sudo password)
```bash
ansible-playbook -i inventory/ --become slapd.yml --user root --connection=local -K
```
3. Some systems can block LDAP starting procesure, like AppArmor. For this specific case, follow these steps.
    - try to run the playbook without modyfing anything
    - if _Start the OpenLDAP Server_ task fails, then continue with the next steps
    - check ```systemctl status slapd.service``` for _unable to open pid file "/var/run/ldap/slapd.pid": 13 (Permission denied)_
    - if that is indeed the issue, check also ```sudo dmesg | grep DENIED | grep slapd```
    - if there are DENIED entries: ```sudo nano /etc/apparmor.d/usr.sbin.slapd``` and add this line:
      **/run/ldap/slapd.pid rwk,**
    - then run ```sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.slapd```
    - restart slapd: ```sudo systemctl restart slapd```
    - run just the remaining _Create entities_ task from main.yml

# Check status
```bash
systemctl status slapd.service
```

# Logs
```bash
journalctl -u slapd
```
---

# LDAP Data Migration Guide

This guide can be used for both **QA** and **PROD** environments by adjusting the hostnames and file paths accordingly.  

---

## 1. Backup from Old VM

```bash
# SSH into the old LDAP VM
ssh root@<OLD_LDAP_HOST>

# Backup OpenLDAP data
slapcat -l /root/today.ldif

# Copy backup file to the new VM
scp root@<OLD_LDAP_HOST>:/root/today.ldif root@<NEW_LDAP_HOST>:/root/
```

## 2. Restore on New VM

```bash
# SSH into the new LDAP VM
ssh root@<NEW_LDAP_HOST>

# Stop the running LDAP service
systemctl stop slapd.service

# Backup existing LDAP directory and create a new one
cd /var/lib/
mv ldap/ ldap.$(date +%Y%m%d)
mkdir ldap

# Restore from the LDIF file
slapadd -f /etc/ldap/slapd.conf -l ~/today.ldif

# Fix ownership permissions
chown -R openldap:openldap ldap

# Restart LDAP service
systemctl start slapd.service
```

## Notes

* Replace `<OLD_LDAP_HOST>` and `<NEW_LDAP_HOST>` with the appropriate hostnames or IPs for your environment.
* Ensure you have sufficient privileges (root or sudo) on both systems.
* Always test restoration on QA before applying to PROD.
