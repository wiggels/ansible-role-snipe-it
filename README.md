Snipe-IT
=========

Ansible role to install, configure, and update a Snipe-IT server on Ubuntu

Requirements
------------

This role currently supports Ubuntu 20.04 and 22.04.

Role Variables
--------------

todo

Dependencies
------------

nginxinc.nginx_core collection

Example Playbook
----------------

    - hosts: all
      tasks:
        - name: Configure port forwarding
          ansible.builtin.include_role: 
            name: wiggels.snipeit
          vars:
            - snipe_domain: snipeit.example.com
            - snipe_db_password: secret
            - snipe_mail_host: smtp.example.com
            - snipe_mail_from_addr: snipe-it@example.com

License
-------

MIT

Author Information
------------------

Hunter Wigelsworth
[GitHub](https://github.com/wiggels)
[LinkedIn](https://www.linkedin.com/in/wiggels/)
[Twitter](https://www.twitter.com/wiggels/)
[Facebook](https://www.facebook.com/wiggels/)
