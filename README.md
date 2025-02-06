## systemd timers
- systemd is the default init system on most modern Linux distros
	- It is the first process (PID 1) started by the kernel which spawns all other userspace processes on the system
	- It is also a powerful automation utility that can be used to start and run daemons and scripts with specific configurations
- systemd operates on objects called units, whose configuration is defined in unit files located under `/etc/systemd
- There are different types of systemd units
	- Services are scripts or programs that systemd can run and their configuration (including the path to their source, or what command to run, or how to run them) is defined in unit files ending in `.service`
	- Timers are units that define how to run a pre-defined `.service` on a particular schedule, and they are defined in unit files ending in `.timer`
## systemd timers vs cron
- While cron can be used to run scripts on a schedule, you have to write all of the logic for logging and monitoring into the script itself
	- You have to decide where your script will write its logs to for example, in case you ever want to view them
- With systemd, you can check the status of a service or timer with `systemctl status <timer-or-service>` and you can  `journalctl -u <timer-or-service>`
## Automating updates with a systemd timer
- Instead of manually running `sudo apt update -y && sudo apt upgrade -y && flatpak update -y`, you can just create a systemd service that runs this command and then a systemd timer to run the service on a specific schedule
- You can create a systemd timer that runs this command daily
### Creating the service
- Create a unit file for a systemd service that wraps the update command
	- `Type=oneshot` means this service is attempted to run only once
	- `ExecStart` defines the command to run
	- `User` must be set to `root` because `apt` needs to be run as root to perform updates
- After creating the unit file, run `sudo systemctl daemon-reload` to reload systemd's configuration (so that it is aware of the newly added service file)
- You can do a test run of the service with `sudo systemctl start daily_update.service` and you can view its status with `sudo systemctl status daily_update.service`
- You can view logs from it with `journalctl -u daily_update.service` and tail them in real time with the `-f` option
```
 $ sudo vim /etc/systemd/system/daily_update.service
 
 $ cat /etc/systemd/system/daily_update.service 
[Unit]
Description=Updates system and application packages daily.

[Service]
Type=oneshot
ExecStart=/bin/bash -c "sudo apt update -y && sudo apt upgrade -y && flatpak update -y"
User=root

$ sudo systemctl daemon-reload

$ sudo systemctl start daily_update.service

$ sudo systemctl status daily_update.service

$ journalctl -u daily_update.service

$ journalctl -u daily_update.service -f
^C
```
### Creating the timer
- Create a unit file for a systemd timer that runs the `daily_update` service that was previously created
	- `OnBootSec` determines the amount of time after boot to wait to run the service
		- In this case, we are giving the system and its daemons/resources 10 minutes head start to finish initializing before we update
	- `OnUnitActiveSec` determines how frequently to run the service, in this case every 24 hours
		- As currently configured, systemd will not wake the computer up just to run the service, nor will it run it multiple times to try to "catch up" in case it missed a few days because the computer was off
		- It will simply try to run the service 24 hours after the last time it was run
	- `WantedBy` must be set to `timers.target` so that systemd knows to run this as a timer
		- `targets` are the various modes or configurations or groupings of units that systemd can run
- After creating the timer, run `sudo systemctl daemon-reload` to update systemd's configuration so systemd is aware of it
- Then run `sudo systemctl enable daily_update.timer` and `sudo systemctl start daily_update.timer` to start the timer
- You can view the status of the timer with `sudo systemctl status daily_update.timer` and logs with `journalctl -u daily_update.timer`
```

$ sudo vim /etc/systemd/system/daily_update.timer

$ cat /etc/systemd/system/daily_update.timer
[Unit]
Description=Timer for running daily system and application package updates

[Timer]
OnBootSec=10min
OnUnitActiveSec=24h
Unit=daily_update.service

[Install]
WantedBy=timers.target

$ sudo systemctl daemon-reload

$ sudo systemctl enable daily_update.timer

$ sudo systemctl start daily_update.timer

$ systemctl status daily_update.timer

```
