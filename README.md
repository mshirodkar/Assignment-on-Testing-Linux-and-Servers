# Assignment-on-Testing-Linux-and-Servers
Assignment on Testing, Linux and Servers
Task 1: System Monitoring Setup

Objective: Configure monitoring of CPU, memory, processes, disk usage; log metrics; document setup.

Implementation steps:

Install a monitoring tool — for example htop (or nmon).

sudo yum install -y htop        # on RHEL/CentOS  
# or sudo apt install -y htop   # on Debian/Ubuntu


Might optionally install nmon, but let’s assume htop is used.

Set up regular snapshots/logging of system metrics. Example: create simple cron job or script that logs top, free, df, du, etc.

Create a directory for logs, e.g. /var/log/sysmon/

sudo mkdir -p /var/log/sysmon
sudo chown root:root /var/log/sysmon
sudo chmod 700 /var/log/sysmon


Create a script /usr/local/bin/sysmon_log.sh:

#!/bin/bash
LOGDIR="/var/log/sysmon"
TIMESTAMP=$(date +'%Y-%m-%d_%H%M')
# CPU/memory snapshot  
top -b -n1 > "$LOGDIR/top_$TIMESTAMP.log"
# free memory  
free -h > "$LOGDIR/free_$TIMESTAMP.log"
# disk usage  
df -h > "$LOGDIR/df_$TIMESTAMP.log"
# large directories usage (example)  
du -h /var/www/html | sort -rh | head -20 > "$LOGDIR/du_wwwhtml_$TIMESTAMP.log"


Make the script executable:

sudo chmod +x /usr/local/bin/sysmon_log.sh


Schedule the logging script via cron. For example, every hour.

sudo crontab -e
# add:
0 * * * * /usr/local/bin/sysmon_log.sh


(This logs at minute 0 each hour.)

Process monitoring: Use htop on-demand for interactive view, and you can also grep the logs for resource-intensive processes. For example you might add to the script something like:

ps aux --sort=-%mem | head -10 > "$LOGDIR/topmem_$TIMESTAMP.log"
ps aux --sort=-%cpu | head -10 > "$LOGDIR/topcpu_$TIMESTAMP.log"


Documentation: 
Installation steps of htop (or nmon)

Explanation of what each part of the logging script does (top, free, df, du)

Cron schedule, location of log files

Example screenshot/terminal output from htop and sample log files (showing e.g. high memory processes)

Observations: if you saw any particular processes consuming high CPU/memory or disk usage trends.

Capacity-planning note: show how the logs can be used for trend analysis (e.g., disk growth, memory usage over time).

Why this makes sense:

Using htop gives interactive visibility; df, du give disk‐usage data.

Logging periodically gives historical data for troubleshooting intermittent performance issues.

Script + cron = automated, consistent.

Documentation ensures the setup is repeatable and auditable.

Task 2: User Management and Access Control

Objective: Create user accounts for Sarah and Mike, set secure passwords, create isolated directories with correct permissions, enforce password policy (expiration, complexity).

Implementation steps:

Create user accounts:

sudo useradd -m -d /home/Sarah -s /bin/bash Sarah
sudo useradd -m -d /home/mike -s /bin/bash mike


(Note: ensure correct case for home directory names; consistent.)

Set secure passwords: You could set initial passwords and force change on first login:

sudo passwd Sarah
sudo passwd mike
sudo chage -d 0 Sarah     # force password change at next login  
sudo chage -d 0 mike


Use strong passwords (document in your report that you used a random strong password and shared securely).

Create isolated working directories:

sudo mkdir -p /home/Sarah/workspace
sudo mkdir -p /home/mike/workspace
sudo chown Sarah:Sarah /home/Sarah/workspace
sudo chown mike:mike /home/mike/workspace
sudo chmod 700 /home/Sarah/workspace
sudo chmod 700 /home/mike/workspace


This ensures only the user can access their workspace.

Enforce password policy (complexity + expiration)

Install or ensure pam_pwquality or pam_cracklib is available. Example on Ubuntu: sudo apt install libpam-pwquality. 
LinuxTechi
+2
Baeldung on Kotlin
+2

Edit /etc/pam.d/common-password (Debian/Ubuntu) or /etc/pam.d/system-auth (RHEL) to require e.g. minlen=12, ucredit=-1, lcredit=-1, dcredit=-1, ocredit=-1. 
Ask Ubuntu
+1

Edit /etc/login.defs to set PASS_MAX_DAYS to 30 days (so passwords expire every 30 days). 
LinuxTechi
+1

Example changes:

# in /etc/login.defs
PASS_MAX_DAYS   30
PASS_MIN_DAYS   1
PASS_WARN_AGE   7


You can also enforce password history so users cannot reuse old passwords: by setting remember=10 in /etc/security/pwquality.conf or similar. 
Red Hat Docs
+1

Documentation: 

Commands to create users, set passwords.

Screenshots/terminal output showing creation and directory permissions.

Explanation of directory permissions (why 700, why owned by user).

Steps to configure password policy with PAM and login.defs (what exactly you changed, file names).

Sample chage -l Sarah output showing password expiration date.

Any challenges (for example if pam_pwquality package had to be installed, or if existing users needed migration).

Note on least privilege: only the user owns their workspace; no unnecessary sudo access unless required.

Task 3: Backup Configuration for Web Servers

Objective: Automate backups for Sarah’s Apache server (config + document root) and Mike’s Nginx server (config + document root); schedule cron jobs every Tuesday at midnight; naming convention; verify integrity; log actions.

Implementation steps:

Ensure backup destination directory exists:

sudo mkdir -p /backups
sudo chown root:root /backups
sudo chmod 700 /backups


(Alternatively you may create sub‐directories per server: /backups/apache/, /backups/nginx/, but the spec says use /backups/.)

Backup scripts:

For Sarah’s Apache server: create script /usr/local/bin/backup_apache.sh

#!/bin/bash
DATE=$(date +'%F')
BACKUP_DIR="/backups"
FILENAME="apache_backup_${DATE}.tar.gz"
SRC1="/etc/httpd"
SRC2="/var/www/html"
cd /
tar -czf "${BACKUP_DIR}/${FILENAME}" "${SRC1}" "${SRC2}"
# verify integrity: list contents  
tar -tzf "${BACKUP_DIR}/${FILENAME}" > "${BACKUP_DIR}/verify_apache_${DATE}.log"


Make executable: chmod +x /usr/local/bin/backup_apache.sh

For Mike’s Nginx server: create /usr/local/bin/backup_nginx.sh

#!/bin/bash
DATE=$(date +'%F')
BACKUP_DIR="/backups"
FILENAME="nginx_backup_${DATE}.tar.gz"
SRC1="/etc/nginx"
SRC2="/usr/share/nginx/html"
cd /
tar -czf "${BACKUP_DIR}/${FILENAME}" "${SRC1}" "${SRC2}"
# verify integrity: list contents  
tar -tzf "${BACKUP_DIR}/${FILENAME}" > "${BACKUP_DIR}/verify_nginx_${DATE}.log"


Ensure the verify logs are generated, owned appropriately, you may include timestamp in logs, etc.

Cron job scheduling: since requirements say “every Tuesday at 12:00 AM” (midnight). In crontab:

sudo crontab -e
# Add:
0 0 * * 2 /usr/local/bin/backup_apache.sh
0 0 * * 2 /usr/local/bin/backup_nginx.sh


This means at minute 0, hour 0 (midnight), every Tuesday (day-of-week 2) run the scripts.

Naming convention and location check: backup files should appear in /backups/ such as apache_backup_2025-10-26.tar.gz, nginx_backup_2025-10-26.tar.gz and verification logs verify_apache_2025-10-26.log, etc.

Integrity verification: The tar -tzf lists the contents of the archive without extracting, documenting that the archive is readable. Additional step: you might capture error code and send email or log if non-zero. Example in script:

if tar -tzf "${BACKUP_DIR}/${FILENAME}" > /dev/null ; then
  echo "Backup ${FILENAME} verification succeeded at $(date)" >> "${BACKUP_DIR}/backup_log.log"
else
  echo "Backup ${FILENAME} verification FAILED at $(date)" >> "${BACKUP_DIR}/backup_log.log"
fi


The listing and return‐code check is aligned with guidance. 
Red Hat
+1

Documentation: In your report include:

The scripts (show full content) for both Apache and Nginx backups.

Cron configuration lines (screenshot or snippet).

Sample backup files in /backups/ with correct naming.

Sample verification logs (verify_*.log) and/or the backup_log.log.

Statement of how integrity is verified and what you would do upon failure.

Challenges (e.g., permissions, disk space, scheduling, testing run).

Advice on retention policy (though not required, you could note that old backups should be purged or archived, aligning with best practices). 
Medium
+1

