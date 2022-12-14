tp.zabbix_ldap_users
====================

Create user-groups that authenticate via ldap and grant them access to specific hostgroups.

Most of our applications have themselves, a db, and related proxies.
It's probably better to put this role in the dependencies of the application,
than to call it from all the parts individually.
That way, you create a user_group that has access to all relevant hosts in Zabbix.
This allows us fine-grained access-control IF we are using AD-security-groups properly.

Example Playbook
----------------

    - name: Create or update user
      hosts: localhost

      roles:
        - role: tp.zabbix_ldap_users
          tp_zabbix_ldap_users_service_name: monitoring
          tp_zabbix_ldap_users_security_group: sec_tg_cs_monitoring_none
          tp_zabbix_ldap_users_rights:
            - hostgroup: monitoring
              permission: read-only
            - hostgroup: monitoring-server
              permission: read-only
            - hostgroup: monitoring-proxy
              permission: read-only
