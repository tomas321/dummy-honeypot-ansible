# dummy-honeypot-ansible

Dummy honeypot contains:
- apache2 server
- mariadb
  - apache authetication table
- local rsyslog configuration for forwarding logs
  - requires a remote syslog ip address
- remote rsyslog configuration to listen to goreign connections

## Example playbook

    - hosts: honeypot.server..sk
      roles:
        - role: dummy-honeypot-ansible/ansible-role-mariadb
          vars:
            mariadb_root_password: "masterRootPa$$"
            mariadb_databases:
              - name: production
                init_script: "users.sql"
            mariadb_users:
              - name: apache
                password: apache
                priv: "prod.users:SELECT"
                host: localhost
      post_tasks:
        - name: ensure MariaDB is logging general logs
          blockinfile:
            path: /etc/mysql/mariadb.conf.d/50-server.cnf
            block: |
              general_log_file        = /var/log/mysql/mysql.log
              general_log             = 1
          register: mariadb_logging

        - name: restart mariadb
          become: true
          when: mariadb_logging.changed
          service:
            name: mariadb
            state: restarted

    - hosts: honeypot.server.sk
      roles:
        - role: dummy-honeypot-ansible/apache
          vars:
            mariadb_auth:
              database_name: "production"
              username: apache
              password: apache

    - hosts: rsyslog.server.sk
      roles:
        - dummy-honeypot-ansible/rsyslog_server

    - hosts: honeypot.server.sk
      roles:
        - role: dummy-honeypot-ansible/rsyslog_client
          vars:
            rsyslog_server_hostname: "10.22.34.12"

