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
