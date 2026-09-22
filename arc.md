sudo nano /etc/resolv.conf

nameserver 8.8.8.8
nameserver 1.1.1.1

=========================================
OLD
Cacti - Version 0.8.8h

Server version: Apache/2.4.6 (CentOS)

PHP 5.4.16 (cli) (built: Apr  1 2020 04:07:17) 
Copyright (c) 1997-2013 The PHP Group
Zend Engine v2.4.0, Copyright (c) 1998-2013 Zend Technologies

mysql  Ver 15.1 Distrib 5.5.68-MariaDB, for Linux (x86_64) using readline 5.1

====================

New Plan : 

Ubuntu 24.04
Apache 2.4
PHP 8.3
MariaDB 10.11
Cacti 1.2.x latest

=============================

Ubuntu 24.04
        |
        |
Apache 2.4
        |
PHP 8.3
        |
MariaDB 10.11
        |
Cacti 1.2.x
        |
Restore old database
        |
Copy old RRD files

==============================
