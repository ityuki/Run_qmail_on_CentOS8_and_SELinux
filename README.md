How to run qmail on SELinux
====

* This setting is a sample.
* There is no guarantee.
* Information is provided by AS IS.
* I'm not good at English. sorry.
* If you find a mistake, please write it in the issue

## Target distribution
* CentOS 8

## preconditoon
* CentOS 8 is running.
* qmail installed. (`./setup setup check` is finished)
* The setting of `control/` files completed.
    * Probably the minimum required files are:
        * `/var/qmail/control/me`
        * `/var/qmail/control/defaultdomain`
        * `/var/qmail/control/rcpthosts`

* SELinux tools installed. 
  if not installed, run next yum command.
```sh
yum install setools-console selinux-policy-devel
```

## environment
* qmail install path: `/var/qmail`

----

## Set fcontext /var/qmail
```sh
restorecon -RFv /var/qmail
```

## qmail-start on systemd
* `/etc/systemd/system/qmail-start.service`
```conf
[Unit]
Description=qmail server daemon
After=syslog.target network.target auditd.service

[Service]
Environment="PATH=/var/qmail/bin:/bin:/usr/bin:/usr/local/bin"
ExecStart=/var/qmail/bin/qmail-start ./Maildir/ splogger qmail
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=always
SyslogIdentifier=qmail-start
SyslogFacility=daemon
SyslogLevel=info
SELinuxContext=system_u:system_r:qmail_start_t:s0

[Install]
WantedBy=multi-user.target
```

### Allows qmail-start on SELinux
* make `qmail_splogger_t.te` file
```conf
module qmail_splogger_t 1.0;
require {
        type devlog_t;
        type qmail_splogger_t;
        type init_var_run_t;
        type syslogd_var_run_t;
        type kernel_t;
        class lnk_file read;
        class sock_file write;
        class unix_dgram_socket sendto;
        class dir search;
}

allow qmail_splogger_t devlog_t:lnk_file read;
allow qmail_splogger_t devlog_t:sock_file write;
allow qmail_splogger_t init_var_run_t:dir search;
allow qmail_splogger_t kernel_t:unix_dgram_socket sendto;
allow qmail_splogger_t syslogd_var_run_t:dir search;
```
* install module
```sh
checkmodule -M -m -o qmail_splogger_t.mod qmail_splogger_t.te
semodule_package -o qmail_splogger_t.pp -m qmail_splogger_t.mod
semodule -i qmail_splogger_t.pp
```


* make `qmail_clean_t.te` file
```conf
module qmail_clean_t 1.0;
require {
        type qmail_clean_t;
        type qmail_spool_t;
        class dir { unlink read rw_dir_perms };
        class file { unlink read getattr };
}
allow qmail_clean_t qmail_spool_t:dir { unlink read rw_dir_perms };
allow qmail_clean_t qmail_spool_t:file { unlink read getattr };
```
* install module
```sh
checkmodule -M -m -o qmail_clean_t.mod qmail_clean_t.te
semodule_package -o qmail_clean_t.pp -m qmail_clean_t.mod
semodule -i qmail_clean_t.pp
```


* make `qmail_lspawn_t.te` file
```conf
module qmail_lspawn_t 1.0;

require {
        type qmail_lspawn_t;
        type var_t;
        class file { open read };
}
allow qmail_lspawn_t var_t:file { open read };
```
* install module
```sh
checkmodule -M -m -o qmail_lspawn_t.mod qmail_lspawn_t.te
semodule_package -o qmail_lspawn_t.pp -m qmail_lspawn_t.mod
semodule -i qmail_lspawn_t.pp
```


* make `qmail_queue_t.te` file
```conf
module qmail_queue_t 1.0;

require {
        type qmail_queue_t;
        type qmail_start_t;
        class fifo_file { read write };
}
allow qmail_queue_t qmail_start_t:fifo_file { read write };
```
* install module
```sh
checkmodule -M -m -o qmail_queue_t.mod qmail_queue_t.te
semodule_package -o qmail_queue_t.pp -m qmail_queue_t.mod
semodule -i qmail_queue_t.pp
```


* run `qmail-start` and check status
```sh
systemctl start qmail-start
systemctl status qmail-start
```
* If status have not error, to enable service
```sh
systemctl enable qmail-start
```

## qmail-smtpd on systemd
* make `/etc/systemd/system/qmail-smtpd.socket` file
```conf
[Unit]
Description=Stream socket for qmail-smtpd server

[Socket]
ListenStream=0.0.0.0:25
Accept=yes

[Install]
WantedBy=sockets.target
```
* make `/etc/systemd/system/qmail-smtpd@.service` file
```conf
[Unit]
Description=qmail-smtpd server
Requires=qmail-smtpd.socket
After=qmail-smtpd.socket

[Service]
ExecStart=/var/qmail/bin/tcp-env /var/qmail/bin/qmail-smtpd
StandardInput=socket
StandardOutput=socket
User=qmaild
Group=nofiles
SELinuxContext=system_u:system_r:inetd_t:s0

[Install]
WantedBy=qmail-smtpd.socket
```


### Allows qmail-smtpd on SELinux
* make `qmail_tcp_env_exec_t.te` file
```conf
module qmail_tcp_env_exec_t 1.0;

require {
        type qmail_tcp_env_exec_t;
        type qmail_tcp_env_t;
        type inetd_t;
        type init_t;
        type node_t;
        class file entrypoint;
        class process transition;
        class tcp_socket node_bind;
}
allow init_t inetd_t:process transition;
allow inetd_t qmail_tcp_env_exec_t:file entrypoint;
allow qmail_tcp_env_t node_t:tcp_socket node_bind;
```
* install module
```sh
checkmodule -M -m -o qmail_tcp_env_exec_t.mod qmail_tcp_env_exec_t.te
semodule_package -o qmail_tcp_env_exec_t.pp -m qmail_tcp_env_exec_t.mod
semodule -i qmail_tcp_env_exec_t.pp
```


* make `socket_qmail_tcp_env_t.te` file
```conf
module socket_qmail_tcp_env_t 1.0;

require {
        type smtp_port_t;
        type init_t;
        type qmail_tcp_env_t;
        class tcp_socket { accept bind create getattr ioctl listen name_bind setopt };
}
allow init_t qmail_tcp_env_t:tcp_socket { accept bind create getattr ioctl listen setopt };
allow qmail_tcp_env_t smtp_port_t:tcp_socket name_bind;
```
* install module
```sh
checkmodule -M -m -o socket_qmail_tcp_env_t.mod socket_qmail_tcp_env_t.te
semodule_package -o socket_qmail_tcp_env_t.pp -m socket_qmail_tcp_env_t.mod
semodule -i socket_qmail_tcp_env_t.pp
```

* run `qmail-smtpd@.socket` and check status
```sh
systemctl start qmail-smtpd@.socket
systemctl status qmail-smtpd@.socket
```
* If status have not error, to enable service
```sh
systemctl enable qmail-smtpd@.socket
```

## qmail-start and qmail-smtpd is runnning!
* check error on `/var/log/audit/audit.log`
* probably you can send and receive by SMTP
* qmail-pop3d? I don't use it so I don't know.
    * Let's use Dovecot!

----
Thank you.

copyright (c) 2020 Ituki Kirihara/NI All rights reserved.

