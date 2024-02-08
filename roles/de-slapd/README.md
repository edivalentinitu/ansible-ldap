# de-slapd

Installs the OpenLDAP Server for the CyVerse Discovery Environment

## Role Variables

| Variable             | Default           | Description                                      |
| -------------        | ----------------- | ------------------------------------------------ |
| dn_suffix            | dc=cyverse,dc=org | The suffix to use for DNs in the LDAP directory. |
| root_password        | notprod           | The password to use for the manager account.     |
| de_grouper_password  | notprod           | The password of the `de_grouper` account.        |
| ldap_reader_password | notprod           | The password of the `ldap_reader` account.       |
| ldap_portal_password | notprod           | The password of the `ldap_portal` account.       |

Use of the default role variable values is not recommended for production systems.

## Example Playbook

``` yaml
- hosts: ldap
  roles:
     - role: de-slapd
       vars:
         dn_suffix: dc=example,dc=org
         root_password: notreal
```
