# Setting up the Grandmaster

# Prerequisites

Before following the guide - install linuxptp package.

```bash
sudo apt update
sudo apt upgrade
sudo apt install linuxptp
```

# Set up the ts2phc

First set up the ts2phc that will synchronize eth ports' PHC to the time from the GNSS module

## Create ts2phc service

1. Run your favorite text editor to create the new service for ts2phc

   ```bash
   sudo vim /usr/lib/systemd/system/ts2phc@.service
   ```

   paste the following systemd script

   ```
   [Unit]
   Description=Synchronizes PTP Hardware Clocks using external time stamps for %I
   Documentation=man:ts2phc
   After=sys-subsystem-net-devices-%i.device
   Before=ptp4l@.service

   [Service]
   Type=simple
   ExecStartPre=/bin/sleep 30
   ExecStartPre=/usr/sbin/phc_ctl %I -- set
   ExecStart=/usr/sbin/ts2phc -f /etc/linuxptp/ts2phc.conf -s generic -c %I

   [Install]
   WantedBy=ptp4l@.service
   ```

2. Create the config file

   ```bash
   sudo vim /etc/linuxptp/ts2phc.conf
   ```

   ```
   [global]
   logging_level 7
   clock_servo linreg

   #time stamper follower device
   [eth0]
   ts2phc.channel 0
   ```

3. Reload the services and run it

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start ts2phc@eth0
   ```

4. Verify the service is running:

   ```bash
   sudo  systemctl status ts2phc@eth0
   ```

   ```
   ● ts2phc@eth0.service - Synchronizes one or more PTP Hardware Clocks using external time stamps
        Loaded: loaded (/lib/systemd/system/ts2phc@.service; enabled; vendor preset: enabled)
        Active: active (running) since Wed 2022-10-12 11:57:11 CEST; 29min ago
      Main PID: 1018 (ts2phc)
         Tasks: 1 (limit: 4164)
           CPU: 1.122s
        CGroup: /system.slice/system-ts2phc.slice/ts2phc@eth0.service
                └─1018 /usr/sbin/ts2phc -f /etc/linuxptp/ts2phc.conf -s generic -c eth0

   Oct 12 12:26:21 cm4-4G ts2phc[1018]: [1786.332] eth0 master offset         24 s2 freq   -2536
   Oct 12 12:26:22 cm4-4G ts2phc[1018]: [1787.088] eth0 extts index 0 at 1665570382.000000024 corr 0 src 1665570382.7051002 diff 24
   Oct 12 12:26:22 cm4-4G ts2phc[1018]: [1787.088] linreg: points 4 slope 1.000002533 intercept -26 err 9
   Oct 12 12:26:22 cm4-4G ts2phc[1018]: [1787.088] eth0 master offset         24 s2 freq   -2507
   Oct 12 12:26:23 cm4-4G ts2phc[1018]: [1788.096] eth0 extts index 0 at 1665570382.999999987 corr 0 src 1665570383.14874569 diff -13
   Oct 12 12:26:23 cm4-4G ts2phc[1018]: [1788.096] linreg: points 4 slope 1.000002538 intercept 10 err 9
   Oct 12 12:26:23 cm4-4G ts2phc[1018]: [1788.096] eth0 master offset        -13 s2 freq   -2548
   Oct 12 12:26:24 cm4-4G ts2phc[1018]: [1789.104] eth0 extts index 0 at 1665570383.999999982 corr 0 src 1665570384.22891453 diff -18
   Oct 12 12:26:24 cm4-4G ts2phc[1018]: [1789.104] linreg: points 4 slope 1.000002544 intercept 14 err 9
   Oct 12 12:26:24 cm4-4G ts2phc[1018]: [1789.104] eth0 master offset        -18 s2 freq   -2558
   ```



## Start ptp4l in grandmaster mode

1. Modify the ptp4l.service to start after a period of time to let the phc to synchronize to the external 1PPS signal

   ```bash
   sudo vim /usr/lib/systemd/system/ptp4l@.service
   ```

   ```
   [Unit]
   Description=Precision Time Protocol (PTP) service for %I
   Documentation=man:ptp4l
   After=ts2phc@.service

   [Service]
   Type=simple
   ExecStartPre=/bin/sleep 10
   ExecStart=/usr/sbin/ptp4l -f /etc/linuxptp/ptp4l.conf -i %I

   [Install]
   WantedBy=multi-user.target
   ```

2. Modify the config file

   ```bash
   sudo vim /etc/linuxptp/ptp4l.conf
   ```

   and modify tx timestamp timeout in the [global] section to prevent timeouts when waiting for tx timestamps

   ```
   [Global]
   # Allow more time for tx timestamps to arrive
   tx_timestamp_timeout 100
   # Set clock class to 6 - clock locked to the PRC
   clockClass 6
   # prevent entering SLAVE state
   masterOnly 1
   ...
   ```

   - apply correct clockClass


   | clockClass | meaning                                      |
   | ---------- | -------------------------------------------- |
   | 6          | Locked with Primary Reference Clock (PRC)    |
   | 7          | PRC unlocked but still in spec               |
   | 13         | Locked to app specific timescale             |
   | 14         | Unlocked from app specific time, but in spec |
   | 248        | Default, if nothing else applies             |
   | 255        | Slave Only Clock                             |

3. Reload services and run it

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start ptp4l@eth0
   ```

4. Verify the service is running

   Last lines should say `assuming the grand master role`

   ```bash
   sudo systemctl status ptp4l@eth0
   ```

   ```
   ● ptp4l@eth0.service - Precision Time Protocol (PTP) service for eth0
        Loaded: loaded (/lib/systemd/system/ptp4l@.service; enabled; vendor preset: enabled)
        Active: active (running) since Wed 2022-10-12 11:57:21 CEST; 32min ago
          Docs: man:ptp4l
      Main PID: 1185 (ptp4l)
         Tasks: 1 (limit: 4164)
           CPU: 939ms
        CGroup: /system.slice/system-ptp4l.slice/ptp4l@eth0.service
                └─1185 /usr/sbin/ptp4l -f /etc/linuxptp/ptp4l.conf -i eth0

   Oct 12 11:57:11 cm4-4G systemd[1]: Starting Precision Time Protocol (PTP) service for eth0...
   Oct 12 11:57:21 cm4-4G systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
   Oct 12 11:57:21 cm4-4G ptp4l[1185]: [46.249] selected /dev/ptp0 as PTP clock
   Oct 12 11:57:21 cm4-4G ptp4l[1185]: [46.251] port 1: INITIALIZING to LISTENING on INIT_COMPLETE
   Oct 12 11:57:21 cm4-4G ptp4l[1185]: [46.252] port 0: INITIALIZING to LISTENING on INIT_COMPLETE
   Oct 12 11:57:27 cm4-4G ptp4l[1185]: [53.051] port 1: LISTENING to MASTER on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES
   Oct 12 11:57:27 cm4-4G ptp4l[1185]: [53.051] selected local clock e45f01.fffe.c75bc0 as best master
   Oct 12 11:57:27 cm4-4G ptp4l[1185]: [53.051] port 1: assuming the grand master role
   ```

5. Configure firewall to allow the PTP service

   ```bash
   sudo firewall-cmd --add-service=ptp
   sudo firewall-cmd --runtime-to-permanent
   ```

## Enable services autostart

   ```bash
   sudo systemctl enable ts2phc@eth0
   sudo systemctl enable ptp4l@eth0
   ```

