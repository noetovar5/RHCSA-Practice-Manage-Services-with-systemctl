# RHCSA-Practice-Manage-Services-with-systemctl
RHCSA Practice — Manage Services with systemctl
Today’s objective: Create, start, stop, restart, enable, inspect, and troubleshoot a harmless custom systemd service.
Estimated time: 45 minutes
Lab system: RHEL 9 or RHEL 10
Privileges required: Root or sudo
This lesson moves from account administration into operating a running Linux system. Starting, stopping, checking, and enabling services are core RHCSA skills.
Important system-availability warning
systemctl controls system services. Stopping the wrong unit can interrupt networking, SSH, databases, web applications, or the entire system.
This lab uses only a custom service named:
rhcsa-lab.service

Do not substitute production services such as sshd, NetworkManager, firewalld, or database services when practicing stop and restart operations over a remote connection.
1. The concept in plain language — 7 minutes
systemd is the service manager used by RHEL. It starts services during boot, tracks their state, restarts them when requested, and records their output in the system journal.
The main administration command is:
systemctl ACTION UNIT

Common actions include:
Command	Purpose
systemctl status UNIT	Display current status
systemctl start UNIT	Start now
systemctl stop UNIT	Stop now
systemctl restart UNIT	Stop and start again
systemctl reload UNIT	Reload configuration without a full restart, if supported
systemctl enable UNIT	Configure automatic startup
systemctl disable UNIT	Remove automatic startup
systemctl enable --now UNIT	Enable and start
systemctl is-active UNIT	Report whether it is running
systemctl is-enabled UNIT	Report whether it starts automatically
systemctl cat UNIT	Display the effective unit file


Two states are especially important:
- Active describes the service right now.
- Enabled describes whether it is configured to start automatically during boot.
A service can be:
- Active but disabled
- Inactive but enabled
- Active and enabled
- Inactive and disabled
2. Preflight checks — 3 minutes
Enter a root shell:
sudo -i

Confirm:
whoami
systemctl is-system-running

Expected username:
root

systemctl is-system-running usually reports:
running

A lab VM might report degraded because another unrelated unit has failed. That does not necessarily prevent this exercise.
Confirm the service name is unused:
systemctl status rhcsa-lab.service

Expected:
Unit rhcsa-lab.service could not be found.

Check whether a file already exists:
test -e /etc/systemd/system/rhcsa-lab.service \
  && echo "STOP: unit file already exists"

Expected result:
No output

3. Guided practice: understand a unit file — 5 minutes
This section is explicitly guided practice.
Our service will run:
/usr/bin/sleep infinity

This creates a harmless long-running process that consumes virtually no CPU. It gives systemd a process to supervise without affecting a real application.
The unit file will contain three sections:
[Unit]
Description=RHCSA harmless service-management lab
After=basic.target

[Service]
Type=simple
ExecStart=/usr/bin/sleep infinity

[Install]
WantedBy=multi-user.target

What each directive means:
- Description explains the unit’s purpose.
- After=basic.target controls startup ordering.
- Type=simple means the started process is the service.
- ExecStart contains the exact executable and arguments.
- WantedBy=multi-user.target identifies where the enable link should be connected.
Use absolute paths in ExecStart.
4. Create and validate the unit — 6 minutes
System-service warning
The next commands create a new system-level unit file. They do not alter any existing unit.
Create it:
printf '%s\n' \
'[Unit]' \
'Description=RHCSA harmless service-management lab' \
'After=basic.target' \
'' \
'[Service]' \
'Type=simple' \
'ExecStart=/usr/bin/sleep infinity' \
'' \
'[Install]' \
'WantedBy=multi-user.target' \
> /etc/systemd/system/rhcsa-lab.service

Set standard ownership and permissions:
chown root:root /etc/systemd/system/rhcsa-lab.service
chmod 644 /etc/systemd/system/rhcsa-lab.service

Inspect it:
cat -n /etc/systemd/system/rhcsa-lab.service
stat -c '%A %a %U %G %n' /etc/systemd/system/rhcsa-lab.service

Expected mode:
-rw-r--r-- 644 root root

Validate the unit:
systemd-analyze verify /etc/systemd/system/rhcsa-lab.service

Expected:
No output

No output normally means no validation problems were found.
5. Reload systemd and examine the initial state — 5 minutes
Service-manager warning
The next command makes systemd reread unit definitions. It does not restart running services.
systemctl daemon-reload

Why it matters: after creating or changing a unit file, systemd must reload its configuration before reliably using the new definition.
Display the loaded unit:
systemctl cat rhcsa-lab.service

Check both states:
systemctl is-active rhcsa-lab.service
systemctl is-enabled rhcsa-lab.service

Expected:
inactive
disabled

The nonzero exit statuses from these checks are normal because the unit has not been started or enabled yet.
6. Start and inspect the service — 6 minutes
System-availability warning
Starting this specific custom service is safe. When working with real services, first understand the application impact and dependencies.
Start it:
systemctl start rhcsa-lab.service

Check its state:
systemctl status rhcsa-lab.service --no-pager

Expected important lines:
Loaded: loaded
Active: active (running)

Verify in a script-friendly way:
systemctl is-active rhcsa-lab.service
echo "Exit status: $?"

Expected:
active
Exit status: 0

Find the supervised process:
systemctl show rhcsa-lab.service \
  --property=MainPID \
  --property=ActiveState \
  --property=SubState

Expected pattern:
MainPID=NUMBER
ActiveState=active
SubState=running

Confirm the process:
main_pid=$(systemctl show -p MainPID --value rhcsa-lab.service)
ps -p "$main_pid" -o pid,user,comm,args

You should see a root-owned sleep infinity process.
7. Enable, restart, and inspect logs — 6 minutes
Enable automatic startup:
systemctl enable rhcsa-lab.service

Expected message indicates creation of a symbolic link under a target’s .wants directory.
Verify both states:
systemctl is-active rhcsa-lab.service
systemctl is-enabled rhcsa-lab.service

Expected:
active
enabled

Notice that enable did not need to start the service because it was already running. Ordinarily, enable controls future boot behavior, not current state.
Record the current PID:
old_pid=$(systemctl show -p MainPID --value rhcsa-lab.service)
echo "Old PID: $old_pid"

Brief-availability warning
restart briefly stops and starts a service. For a production application, this can cause an outage. It is safe for this harmless lab service.
systemctl restart rhcsa-lab.service

Capture the new PID:
new_pid=$(systemctl show -p MainPID --value rhcsa-lab.service)
echo "New PID: $new_pid"

The new PID should differ from the old one, proving that the process was replaced.
Inspect journal entries for only this unit:
journalctl -u rhcsa-lab.service -n 10 --no-pager

Useful options:
- -u filters by unit.
- -n 10 shows the last ten entries.
- --no-pager prints directly to the terminal.
8. Stop, start, and test combined operations — 4 minutes
Availability warning
Stopping a real service makes it unavailable. The following command affects only rhcsa-lab.service.
systemctl stop rhcsa-lab.service

Verify:
systemctl is-active rhcsa-lab.service
systemctl is-enabled rhcsa-lab.service

Expected:
inactive
enabled

This demonstrates that stopping does not disable automatic startup.
Now disable and start it:
systemctl disable rhcsa-lab.service
systemctl start rhcsa-lab.service

Verify:
systemctl is-active rhcsa-lab.service
systemctl is-enabled rhcsa-lab.service

Expected:
active
disabled

Finally, practice the combined command:
systemctl enable --now rhcsa-lab.service

Verify:
systemctl is-active rhcsa-lab.service
systemctl is-enabled rhcsa-lab.service

Expected:
active
enabled

9. Independent challenge — 7 minutes
Create and manage a second unit without copying the first unit commands.
Assignment
Create:
/etc/systemd/system/rhcsa-report.service

Requirements:
1. Description:
RHCSA report-generation lab

2. Use:
Type=oneshot

3. Its ExecStart must run /usr/bin/bash and create:
/tmp/rhcsa-service-lab/report.txt

4. The report must contain:
RHCSA service report generated

5. Include:
RemainAfterExit=yes

6. Include an [Install] section associated with multi-user.target.
7. Set the unit owner to root:root and its mode to 0644.
8. Validate it with systemd-analyze verify.
9. Reload the unit definitions.
10. Enable and start it in one operation.
11. Verify:
    - The unit is active.
    - The unit is enabled.
    - The report exists.
    - The report contains the required text.
12. Stop the unit and verify it becomes inactive.
13. Start it again and inspect its journal.
A oneshot service normally performs a task and exits. RemainAfterExit=yes allows systemd to keep the unit in an active state after the command completes successfully.
10. Independent verification
Validate the unit:
systemd-analyze verify /etc/systemd/system/rhcsa-report.service

Check its configuration:
systemctl cat rhcsa-report.service

Check its states:
systemctl is-active rhcsa-report.service
systemctl is-enabled rhcsa-report.service

Both should report:
active
enabled

Verify the result:
test -f /tmp/rhcsa-service-lab/report.txt \
  && echo "PASS: report exists" \
  || echo "FAIL: report missing"

grep -qx 'RHCSA service report generated' \
  /tmp/rhcsa-service-lab/report.txt \
  && echo "PASS: report contents are correct" \
  || echo "FAIL: report contents are incorrect"

Inspect the journal:
journalctl -u rhcsa-report.service -n 10 --no-pager

11. Knowledge check
Answer these without reviewing the lesson:
1. What is the difference between an active and an enabled service?
2. Does systemctl enable necessarily start a service immediately?
3. Does systemctl start necessarily configure startup at boot?
4. Which command performs both operations?
5. Why is systemctl daemon-reload required after changing a unit file?
6. What does systemctl status show?
7. Which command is better for a script that only needs a yes-or-no active-state result?
8. What does ExecStart define?
9. Why should ExecStart use an absolute executable path?
10. What does WantedBy=multi-user.target support?
11. Why can restarting a production service be disruptive?
12. How can you view logs for one service?
13. What does Type=oneshot mean?
14. What does RemainAfterExit=yes change?
15. What should you verify after every service-management task on the RHCSA exam?
I would consider this objective mastered when you can distinguish active from enabled, create and validate a unit, manage its runtime and boot states, inspect its process and logs, and cleanly remove it.
Safe rollback and cleanup
System-availability warning
The following commands stop and disable only the two temporary lab services. Verify the unit names carefully.
systemctl disable --now rhcsa-lab.service
systemctl disable --now rhcsa-report.service

Remove only the temporary unit files:
rm -f /etc/systemd/system/rhcsa-lab.service
rm -f /etc/systemd/system/rhcsa-report.service

Tell systemd that the definitions are gone:
systemctl daemon-reload
systemctl reset-failed

Filesystem cleanup warning
Confirm that the path is exactly /tmp/rhcsa-service-lab:
rm -rf /tmp/rhcsa-service-lab

Verify cleanup:
systemctl status rhcsa-lab.service
systemctl status rhcsa-report.service
test ! -e /etc/systemd/system/rhcsa-lab.service \
  && echo "PASS: primary unit removed"
test ! -e /etc/systemd/system/rhcsa-report.service \
  && echo "PASS: challenge unit removed"
test ! -e /tmp/rhcsa-service-lab \
  && echo "PASS: service-lab data removed"

The status commands should report that the units could not be found.
Leave the root shell if you entered it with sudo -i:
exit




Open chat


RHCSA Practice — Configure Privileged Access with sudo Today’s objective: Safely grant a non-root user permission to run specific administrative commands through a validated /etc/sudoers.d/ rule. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: Root or an existing working sudo account This builds directly on your local users and groups lesson. Configuring privileged access is an explicit RHCSA objective. Critical authentication warnSep 27RHCSA Practice — Manage Local Users and Groups Today’s objective: Create and modify local users and groups, assign supplementary group membership, configure password aging, and verify access. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: Root or sudo This builds on your work with permissions and shell commands. Local account and group administration is an explicit RHCSA objective. Important safety warning The commands in this lessonSep 24RHCSA Practice — Process Multiple Files with a Bash for Loop Today’s focused objective: Use a Bash for loop to process multiple command-line arguments and generate a consolidated log report. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: All work stays under /tmp/rhcsa-loop-lab. This practice does not modify networking, storage, boot configuration, authentication, SELinux, firewall rules, or system services. This buSep 22RHCSA Practice — Build a Log-Checking Bash Script Today’s focused objective: Create a simple Bash script that accepts a command-line argument, validates input with an if statement, processes command output, and returns a meaningful exit status. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: This lesson works only under /tmp/rhcsa-script-lab. It does not modify services, networking, storage, boot configuration, authentiSep 20RHCSA Practice — Search Logs with grep and Regular Expressions Today’s focused objective: Use grep, pipes, and regular expressions to locate useful information in configuration files and application logs. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: Everything stays under /tmp/rhcsa-grep-lab. This lesson does not change networking, storage, boot, authentication, SELinux, firewall rules, or system services. This isSep 17RHCSA Practice — Archive and Compress Files with tar Today’s focused objective: Create, inspect, extract, compress, and verify archives using tar, gzip, and bzip2. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneExam relevance: Red Hat’s current EX200 objectives require candidates to archive, compress, unpack, and uncompress files using tar, gzip, and bzip2. [Official RHCSA EX200 objectives](https://www.redhat.com/en/Sep 15RHCSA Practice — Hard and Symbolic Links Today’s focused objective: Create, inspect, troubleshoot, and remove hard links and symbolic links. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10 VMExam relevance: Creating hard and soft links is listed under the current RHCSA EX200 essential-tools objectives. Official Red Hat EX200 objectives This lesson builds on yoSep 13RHCSA Practice — Standard Linux Permissions Today’s objective: Read, set, and verify standard user/group/other permissions using symbolic and numeric notation. Estimated time: 45 minutesLab system: Your RHEL 9 VM is suitable for this practice.Safety: This lab works entirely inside /tmp/rhcsa-permissions-lab. It does not change networking, storage, boot settings, authentication, SELinux, the firewall, or system availability. Red Hat’s current EX200 objectives includSep 10












    To pick up a draggable item, press the space bar.
    While dragging, use the arrow keys to move the item.
    Press space again to drop the item in its new position, or press escape to cancel.
