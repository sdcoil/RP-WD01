This information is from https://web.archive.org/web/20141112135713/http://www.isartor.org/wiki/Securing_your_RavPower_Filehub_RP-WD01

# Securing your RavPower Filehub RP-WD01

Got myself a RavPower FileHub WD01 for the following reasons:

  - Able to share a hotel wifi connection (pay only once and connect all your wifi devices)
  - Able to stream photos and videos from the filehub to PC/Phone/Tablet (Ability to add SDcard/USBstorage to filehub which is shared to connected devices through samba or AirStor Android/IOS App)
  - Next to that you can also use it as a powerbank, as it has an integrated 3000maH battery. 

Playing around with the filehub I noticed that this thing does what it say it does, however the builders didn't think much about securing the device.

The filehub has a telnet daemon running which listens on ALL interfaces. When you connect the filehub to a foreign wifi network, this means that the telnet daemon is exposed to all devices on the network to which you are connecting (the foreign network). Next to that, the "root" password (20080826) is freely avaiable on the internet (found it in multiple forum topics), so it is not very difficult for any "hacker" (or curious individual with some technical knowhow), on the foreign network, to access your device and even get root access. Of course this also means that that "hacker" is able to access all your storage devices which you have attached to the filehub, and even modify or delete that data, which in my opinion should raise some red flags.

Putting in a little effort I was able to make the filehub a bit more secure, by:

  - Changing the root password
  - Do some basic IPv4 filtering
  - Disabling IPv6, as a firewall for IPv6 is not installed on the device. 

The main thing here is making your changes on the squashfs filesystem and then committing those changes to flash, which took a little time to figure that out.

# Change the root password

  - Connect a computer to the filehub's hotspot
  - Telnet to 10.10.10.254 (default IP)
  - Login: root
  - Passwd: 20080826 

  - Issue the command: 

        passwd

  - Choose a new password 

  - Commit changes to flash, by issuing the following command: 

        /usr/sbin/etc_tools p

Now the root password is changed and committed to flash, so it stays changed after reboots.

# Basic IP Filtering

You should already be a lot saver by changing the root password. However I don't want the telnet daemon to be accessible from foreign networks. Better yet, I don't even want my samba shares exposed to those foreign networks. So I created my filter rules quite basic, but effective:

  - Connect a computer to the filehub's hotspot
  - Telnet to 10.10.10.254 (default IP)
  - Login: root
  - Passwd: [the_password_you_set] 

The file "/etc/rc.local" is executed after all services has started. So you need to edit this file. The only text editor I could find on the system was "vi". So if you don't know how "vi" works, you should read up on this first.

  - Edit the file: /etc/rc.local by issuing the following command: 
    
        vi /etc/rc.local

  - At the end of this file, append the following: 

        iface="apcli0"                                    

        # Drop all tcp traffic incomming on iface
        /bin/iptables -A INPUT -p tcp -i ${iface} -j DROP
        # Drop all udp traffic incomming on iface
        /bin/iptables -A INPUT -p udp -i ${iface} -j DROP                                         
                     
        # Fetch IPv6 address on iface                                        
        ipv6_addr=`ifconfig ${iface} | grep inet6 | awk {'print $3'}`
                           
        # No IPv6 filter is installed, so remove IPv6 address on iface
        if [ "${ipv6_addr}" != "" ]; then
          /bin/ip -6 addr del "${ipv6_addr}" dev ${iface}
        fi

  - Save the file, and quit "vi" 

  - Commit changes to flash, by issuing the following command: 

        /usr/sbin/etc_tools p

  - Restart the device. 

Now you added some very basic IP filtering. Whatever you do, watch out with filtering icmp traffic, as the filehub uses icmp to check for connectivity on the foreign network.

Doing an nmap of the IP address exposed on the foreign network (running from a host on foreign network):

    nmap -P0 [IP_ADDR_ON_FOREIGN_NET]

    Starting Nmap 5.21 ( http://nmap.org ) at 2014-03-14 21:23 CET

    Nmap scan report for [IP_ADDR_ON_FOREIGN_NET]
    Host is up (0.082s latency).
    All 1000 scanned ports on [IP_ADDR_ON_FOREIGN_NET] are filtered

On a host on the foreign network I also tested with adding a route to 10.10.10.0/24 to gateway [IP_ADDR_ON_FOREIGN_NET]. As 10.10.10.0/24 is a routable network, this should also be tested. After adding the route I could indeed ping the internal IP of the filehub:

    # ping 10.10.10.254
    PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
    64 bytes from 10.10.10.254: icmp_req=1 ttl=64 time=1.67 ms

But nmap still was not able to see any exposed ports from the foreign network:

    # nmap -P0 10.10.10.254
  
    Starting Nmap 5.21 ( http://nmap.org ) at 2014-03-14 21:25 CET

    Nmap scan report for 10.10.10.254
    Host is up.
    All 1000 scanned ports on 10.10.10.254 are filtered

So now I can access the filehub when connected to the filehub's hotspot, but from the foreign network I am not able to access the filehub on tcp or udp.


# Notes

Please remember that these changes need to be applied probably every time after a firmware update. 
