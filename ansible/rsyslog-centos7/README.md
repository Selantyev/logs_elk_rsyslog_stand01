Role Name
=========

Rsyslog server installation

Requirements
------------

CentOS 7

Role Variables
--------------

packages:
- tree
- vim
- htop
- libselinux-python
- policycoreutils-python
- bash-completion
- telnet

Dependencies
------------

Example Playbook
----------------

- hosts: logs
  roles:
  - rsyslog-centos7

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
