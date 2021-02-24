**Admin application**

This is a Django/Python application located in 
`/opt/sc_radio`


**Broadcaster applications**

This is a Django/Python application located in 
`/var/users/<USERNAME>/app`

*Debugging:*

see if application can start with
`cd /var/users/<USERNAME>/app`

or

`/opt/sc_radio`

for admin app and

`./manage.py shell`

if this command does not crash and ends up showing the console - application is fine, otherwise a traceback with an error description is displayed.
Common issues are related to broken Python dependencies or broken packages or database connection issues.


Software depends on the following system-level services, so make sure all of them are running.


**uWsgi**

uWsgi (https://uwsgi-docs.readthedocs.io/en/latest/) is a container app that runs Broadcaster and Admin Django applications.
Configuration files are available in 

`/etc/supervisor/conf.d/<USERNAME>.conf` - for broadcaster applications

`/etc/supervisor/conf.d/sc_radio.conf` - for admin application

**Supervisord**

This service is a daemon that makes sure all admin and broadcaster uWsgi applications are running and restarts the automatically if they crash.

Debugging:
If some broadcaster is not working try to open its config file in 

`/etc/supervisor/conf.d/<USERNAME>.conf`

and change

`stdout_logfile=/dev/null`

to

`stdout_logfile=/path/to/file.log`

Restart supervisord with

`service supervisord restart`

After that log will have a detailed description in case Django application for that account is broken and not running.


**Music Indexing service**

Used to look for mp3/flac files that users upload via web interface or FTP. It syncronizes files on the file system with the database information, calculates file length, extracts images and so on.
It is using 2 extenral programs:

 1. sox (http://sox.sourceforge.net/) to extract mp3 data from audio file
 2. mp3gain (http://mp3gain.sourceforge.net/) to apply volume normalization.

This service is running every 10 minutes according to this CRON rule:
`*/5 * * * * root /usr/local/bin/content_indexer 1>/dev/null 2>/dev/null`

this line is configured in `/etc/crontab`.

*Debugging:*
A log of this service for every user is located in 

`/var/users/<USERNAME>/log/indexer.log`

Also it is possible to save STDERR/STDOUT of this process by changing CRON rule from 

`*/5 * * * * root /usr/local/bin/content_indexer 1>/dev/null 2>/dev/null`

to

`*/5 * * * * root /usr/local/bin/content_indexer 1>>/path/to/output.log 2>>/path/to/error.log`

Indexing service is using configuration file `/opt/bin/indexer.cfg` and it is using MySQL root password to operate, so if MySQL root password changes - it should be changed in this config file as well.


**Nginx**

Nginx is used to server Django applications for broadcaster and admin interfaces.
Configuration files are located in 

`/etc/nginx/conf.d/<USERNAME>.conf` - for broadcaster app

`/etc/nginx/conf.d/sc_radio.conf` - for admin app

Debugging:
By default all Nginx logs are disabled to save disk space. Enable by changing
```
access_log  /dev/null ;
error_log   /dev/null;
```
to
```
access_log  /path/to/access.log;
error_log   /path/to/error.log;
```
And restarting Nginx with

`service nginx restart`

Also see system logs.


**MySQL**

MySQL is used to store all the data for broadcaster/admin accounts. Default system-level configuration is uses with default settings.
When MySQL root password changes it should be also updated in:

`/opt/bin/indexer.cfg`

`/opt/bin/utils.ini` under "[MySQL]" section.

*Debugging:*
Most common problem with MySQL is a crash or unable to start, see system logs for details.
Depending on what MySQL version (MariaDB or original MySQL) is installed use corresponding command to restart it:

`service mysql restart`

or

`service mariadb restart`

**ProFTP**

This is a default FTP server running on port 21. It has a default configuration with MySQL extension enabled for broadcaster accounts.
Configuration is located in /etc/proftpd.conf (CentoS)
Restart with 

`service proftpd restart`

**RadioPoint**

This is a core process that runs user radio stream.
Configuration is available in 

`/var/users/<USERNAME>/conf/radiopoint_<SERVER_ID>.conf`

Debugging
Change 

`LOG=0`

to 

`LOG=3`

in config file, so log file from

`LOGPATH`

setting will have a detailed report how radio station is operating.
Process can be started from the broadcaster web interface, or killed using SSH using standard kill command.
It restarts automatically every minute.


**Utilities**

Additional system utilities are run y CRON, configuration from /etc/crontab usually looks like this:

```
1. */5 * * * * root /usr/local/bin/content_indexer 1>/dev/null 2>/dev/null
2. */1 * * * * root python3.6 /opt/bin/sc_stats 1>/dev/null 2>/dev/null
3. */15 * * * * root python3.6 /opt/bin/sc_backup 1>/dev/null 2>/dev/null
4. */1  * * * * root python3.6 /opt/bin/sc_accounts 1>/dev/null 2>/dev/null
5. 0    * * * * root python3.6 /opt/bin/awstats 1>/dev/null 2>/dev/null
6. 30 2 * * 1 root /usr/bin/letsencrypt renew
```

(1) - music indexing service
(2) - utility that goes through every broadcaster account and collects Shoutcast/Icecast listener statistics.
(3) - Backup script userd to create or restore broadcaster accounts backups.
(4) - used to create or delete broadcaster accounts 
(5) - AWSTATS module that generates advances listener reports based on Icecast/Shoutcast access log files.
(6) - Optional, in case SSL certificate is configured.

Debugging:
1. Change 1>/dev/null 2>/dev/null to real stdout/stderr log files to see what utils are doing and for possible issues.
2. Try to run utility by hand via console, for example python3.6 /opt/bin/sc_accounts to see the output for potential issues.
