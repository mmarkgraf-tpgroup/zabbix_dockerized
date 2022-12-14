# zabbix_dockerized

Ansible-roles to install and configure Zabbix 5.2

This was one of my projects while I was working for [Transporeon](https://www.transporeon.com/).

Please note that these roles will not be able to run, as the depend on quite a few roles
developed at [Transporeon](https://www.transporeon.com/) that I am not allowed to publish here.

The roles in this repo will require quite a bit of refactoring in order to be independent of the
not-published roles.

These roles were written for Zabbix 5.2.  You will want to go for Zabbix >=6.2 instead, as that
offers builtin HA for the first time and the ability to attach additional items to discovered hosts.  
That will require some more refactoring, I'm afraid.

All that being said, these roles can show you how to automate Zabbix in a hands-free fashion,
with every deployed vm automatically monitored as you need.

## Todo

- zabbix_check, zabbix_host and zabbix_dummy_host should probably be refactored into a single role
- the item_factory produces way to many redundant api-calls, probably best use templates instead where possible
- get this ready for Zabbix 6
