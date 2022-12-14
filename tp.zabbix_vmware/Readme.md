Monitoring VMware with Zabbix
=============================

So far, monitoring is done with the Builtin Template "VMware".

Discovery of VMware VMs has been disabled, as it interferes with attaching templates from our Ansible-roles.

I've adapted the python-script previously used through Icinga to produce JSON-output ( see <https://github.com/Napsty/check_esxi_hardware/pull/65> ).
So far, I have not been able to attach it to the hypervisors discovered by the VMware template.
I leave it in files for later, as it provides additional informations such as fan-speed, temperatures, voltages and the state of the fuse.
