![Screenshot](https://raw.githubusercontent.com/wiedehopf/graphs1090/screenshots/screenshot1.png)
![Screenshot](https://raw.githubusercontent.com/wiedehopf/graphs1090/screenshots/screenshot2.png)

# graphs1090
![Screenshot](https://raw.githubusercontent.com/wiedehopf/graphs1090/screenshots/messages_24h.png)
Graphs for dump1090-fa (based on dump1090-tools by mutability)

Also works for other dump1090 variants supplying stats.json

## Installation / Update to current version:
```
sudo bash -c "$(wget -q -O - https://raw.githubusercontent.com/wiedehopf/graphs1090/master/install.sh)"
```

## Configuration (optional):
Edit the configuration file to change graph layout options, for example size:
```
sudo nano /etc/default/graphs1090
```
Ctrl-x to exit, y (yes) and enter to save.

Reset configuration to defaults:
```
sudo cp /usr/share/graphs1090/default-config /etc/default/graphs1090
```


## View the graphs:

Click the following URL and replace the IP address with the IP address of the Raspberry Pi you installed combine1090 on.

http://192.168.x.yy/graphs1090

or

http://192.168.x.yy/perf

### Reducing writes to the sd-card (option 1 of 2)

The rrd databases get written to every minute, this adds up to around 100 Megabytes written per hour.
While most modern SD-cards should handle this for 10 or more years easily, you can reduce the amount written if you want to.
Per default linux writes data to disk after a maximum of 30 seconds in the cache.
Increasing this to 10 minutes reduces actual disk writes to around 10 Megabytes per hour.

Don't change this if you handle data on the Raspberry Pi which you don't want to lose the last 10 minutes of.

Increasing this write delay to 10 minutes can be done like this (takes effect after reboot):
```
sudo tee /etc/sysctl.d/07-dirty.conf <<EOF
vm.dirty_ratio = 40
vm.dirty_background_ratio = 30
vm.dirty_expire_centisecs = 60000
EOF
```

Because i don't mind losing data on my Raspberry Pi when it loses power, i have set this to one hour:
```
sudo tee /etc/sysctl.d/07-dirty.conf <<EOF
vm.dirty_ratio = 40
vm.dirty_background_ratio = 30
vm.dirty_expire_centisecs = 360000
EOF
```

### Reducing writes to the sd-card (option 2 of 2)

If you don't mind losing up to 24h of graph data in case of a power loss, you can drastically reduce the writes to the sd-card by graps1090/collect by executing this script:

```
sudo bash /usr/share/graphs1090/git/malarky.sh
```

It changes the the configuration of the systemd service to manage the graph data in /run (memory) and only write it to disk every night.
On reboot / shutdown it's written to disk and the data loaded to /run again when the system boots back up.
Up to 24h of data is lost when there is a power loss.
This is somewhat new, there might be some other possibilities graph data might be lost that i'm not aware of.
So far it's working perfectly for me.


### Non-standard configuration:

If your local map is not reachable at /dump1090-fa or /dump1090, you can edit the following the file to input the URL of your local map:

```/etc/collectd/collectd.conf```

Find this section:

```
<Plugin python>
        ModulePath "/usr/share/graphs1090"
        LogTraces true
        Import "dump1090"
        <Module dump1090>
                <Instance localhost>
                        URL "http://localhost/dump1090-fa"
                </Instance>
        </Module>
</Plugin>
```
And change the URL to where your dump1090 webinterface is located.
After changing the URL, restart collectd:
```
sudo systemctl restart collectd
```

### Resetting the database format

This might be a good idea if you changed from the adsb receiver project graphs and kept the data.
Also if you upgraded at a somewhen July 15th to July 16th 2019. Had a bad setting removing maximum data keeping for some part of the data.

```
sudo bash -c "$(wget -q -O - https://raw.githubusercontent.com/wiedehopf/graphs1090/master/install.sh)"
sudo apt update
sudo apt install -y screen
sudo screen /usr/share/graphs1090/new-format.sh
```

### Reporting issues:

Please include the output for the following commands in error reports:
```
sudo systemctl restart collectd
sudo journalctl --no-pager -u collectd | tail -n40
sudo /usr/share/graphs1090/graphs1090.sh
sudo systemctl restart graphs1090
```
Paste the output into a pastebin: https://pastebin.com/
Then include the link and be sure to also describe the issue and also mention your system (debian / ubuntu / raspbian and RPi vs x86).


### Known bugs:

##### disk graphs with kernel >= 4.19 don't work due to a collectd bug
https://github.com/collectd/collectd/issues/2951

possible sollution: install new collectd version (only on Raspberry pi, if you are using another architecture, this package won't work)
```
wget -O /tmp/collectd.deb http://raspbian.raspberrypi.org/raspbian/pool/main/c/collectd/collectd-core_5.8.1-1.3_armhf.deb
sudo dpkg -i /tmp/collectd.deb
```


### Deinstallation:
```
sudo bash /usr/share/graphs1090/uninstall.sh
```

### nginx configuration:

```
location /graphs1090/graphs/ {
  alias /run/graphs1090/;
}

location /graphs1090 {
  alias /usr/share/graphs1090/html/;
  try_files $uri $uri/ =404;
}
```


### no http config

in collectd.conf:
```
  URL "file:///usr/local/share/dump1090-data"
```
commands:
```
sudo mkdir -p /usr/local/share/dump1090-data
sudo ln -s /run/dump1090-fa /usr/local/share/dump1090-data/data
```


### Backup and Restore (same architecture)

```
cd /var/lib/collectd/rrd
sudo tar cf rrd.tar localhost
cp rrd.tar /tmp
```
Backup this file:
```
/tmp/rrd.tar
```

I'm not exactly sure how you would do that on Windows.
Probably with FileZilla using the SSH/SCP protocol.

Install graphs1090 if you haven't already.

On the new card copy the file to /tmp using FileZilla again.

Then copy it back to its place like this:
```
sudo mkdir -p /var/lib/collectd/rrd/
cd /var/lib/collectd/rrd
sudo cp /tmp/rrd.tar /var/lib/collectd/rrd/
sudo systemctl stop collectd
sudo tar xf rrd.tar
sudo systemctl restart collectd graphs1090
```

This should be all that is required, no guarantees though!

### Backup and Restore (different architecture, for example moving from RPi to x86 or the other way around)

Basically the same procedure as above, but with this difference:

Before doing the backup, run this command:

```
sudo /usr/share/graphs1090/rrd-dump.sh /var/lib/collectd/rrd/localhost/
```

This creates XML files from the database files in the same directory which can be later restored to database files on the target system.

Complete the process described above (backup the folder, then copy it back to its place on the new card).
Now run this command:

```
sudo /usr/share/graphs1090/rrd-restore.sh /var/lib/collectd/rrd/localhost/
```

Again no guarantees, but this should work.
