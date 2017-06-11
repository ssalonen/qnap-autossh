# Set-up auto-ssh tunnel with QNAP

Following the [QPKG-based
method](https://wiki.qnap.com/wiki/Running_Your_Own_Application_at_Startup)
from the QNAP Wiki

1. Install autossh using [Optware IPKG](https://wiki.qnap.com/wiki/Install_Optware_IPKG):

````
ipkg -i autossh
````
2. Mount the flash drive having the atomount

````
mount -t ext4 /dev/mtdblock5 /tmp/config
````

Note that this is for TS-219P II, consult the [wiki](https://wiki.qnap.com/wiki/Running_Your_Own_Application_at_Startup#MTD-based_method) for suitable mount command for your QNAP model.

3. Create dummy package (follow wiki), at the end of the qpkg conf

````
[autorun]
Name = autorun
Version = 0.1
Author = neomilium
Date = 2013-05-06
Shell = /share/MD0_DATA/.qpkg/autorun/autorun.sh
Install_Path = /share/MD0_DATA/.qpkg/autorun
Enable = TRUE
````
4. Create /share/MD0_DATA/.qpkg/autorun/autorun.sh with the following contents
````
#!/bin/sh
AUTORUN_FOLDER=/share/MD0_DATA/.qpkg/autorun
LOGFILE=$AUTORUN_FOLDER/autorun.log
echo Running $AUTORUN_FOLDER/autorun.sh. Current time $(/bin/date --utc), or local $(/bin/date) > $LOGFILE 2>&1

echo Running open_ssh_tunnels.sh >> $LOGFILE 2>&1
${AUTORUN_FOLDER}/open_ssh_tunnels.sh >> $LOGFILE 2>&1
````
5. `chmod +x /share/MD0_DATA/.qpkg/autorun/autorun.sh`
6. Create /share/MD0_DATA/.qpkg/autorun/open_ssh_tunnels.sh with the following contents

````
#!/bin/sh
set -x
# See https://www.everythingcli.org/ssh-tunnelling-for-fun-and-profit-autossh/

ECHO=/bin/echo
SED=/bin/sed
AUTOSSH=/opt/bin/autossh
SCREEN=/usr/sbin/screen
DESTINATION=yoursshuser@your.host.com
LOCAL_PORTS="21 22 8080"

# Forward ports to remote host. Remote port will be 555 + <last two digits of forwarded port>
cd /
# autossh will retry even on failure of first attempt to run ssh.
export AUTOSSH_GATETIME=0
for localport in $LOCAL_PORTS; do
    port_last_two_digits=$($ECHO $localport|$SED 's/.*\(..\)/\1/')
    remote_port=555${port_last_two_digits}
    TERMINFO=/opt/share/terminfo $SCREEN -dmS sshtunnelloota_${localport} \
                sh -c "echo \"SSH port forward -R ${remote_port}:localhost:${localport}.\" \
                	&& $AUTOSSH -N -M 0 -o 'ServerAliveInterval 30' -o 'ServerAliveCountMax 3' -o 'ExitOnForwardFailure yes' \
                        $DESTINATION -T -R *:${remote_port}:localhost:${localport}"
done
````
7. `chmod +x /share/MD0_DATA/.qpkg/autorun/open_ssh_tunnels.sh`

8. Unmount /tmp/config:

````
umount /tmp/config
````

9. Make sure you have `GatewayPorts` enabled on ssh server if you want to listen to all network interfaces