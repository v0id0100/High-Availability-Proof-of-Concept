# Proof of Concept: High Availability Web Service -- by v0id0100

![main_image](images/main_image.png)


## Index:
- [1. What is High Availability?](#1-what-is-high-availability)
- [2. Project Architecture](#2-project-architecture)
- [3. Tools we are going to use](#3-tools-we-are-goint-to-use)
- [4. Example Web Developing](#4-example-of-web-developing)
- [5. Data Sync between nodes](#5-data-sync-between-nodes)
- [6. Implementing Load Balancer](#6-implementing-load-balancer)
- [7. Syncing / Checking status between nodes](#7-syncing--checking-status-between-nodes)
- [8. Security and Firewall](#8-security-and-firewall)
- [9. Proof of Concept](#9-proof-of-concept)
- [10. Analyzing Improvements and Point of Failure](#10-analyzing-improvements-and-point-of-failure)
- [11. Who am I?](#11-who-am-i)
- [12. Greetings to](#12-greetings-to)

---

## 1. What is High Availability?
High Availability is the definition that a **server o service is 100% accessible and operational of the time**, at least minimizing the downtime.

It is very used nowadays for social media, banks and another kind of applications.

### Why this PoC is High Availability?
Well in this project I will implement more a web cluster, set of nodes working all together, the client only see one web portal, but behind, are 2 nodes (only to proof the concept) working and serving a web portal at the same time.

- Trying to eliminate the **Single Points of Failure**, those are the points that if one of them falls, it would cause a failure in the whole system.

- Implementing **failover**: If one nodes fails, the other is going to be deployed immediately without the client perception because the DNS (IP in the case) never changes.

- For this I'm going to implement a tool called ***conntrackd*** that what it makes is checking every time and asking if their *mate* is still alive.

---

## 2. Project Architecture:
| ROL | HOSTNAME | IP | Function|
|-----|----------|----|---------|
| FW/LB Master | Lb01 | 192.168.1.10 | Active Node, it manage the VIP |
| FW/LB Backup | Lb02 | 192.168.1.11 | Passive Node, it syncs the correct functionality |
| Web Server | Web01 | 192.168.1.21 | Apache Web Service |
| Web Server | Web01 | 192.168.1.22 | Apache Web Service |
| Virtual IP (VIP) | - - - - - - - - - - - - | 192.168.1.100 | Flex IP address, entry point |

---

## 3. Tools we are going to use:
In this PoC we need some requisites before proceed:
- For web service in **Web01, Web02** nodes:
    ```bash
    sudo apt install apache2 php -y
    ```
- To sync data between nodes in **Lb01, Lb02** nodes:
    ```bash
    # In lb02 (Backup)
    sudo apt install openssh-server
    # In lb01 (Master)
    sudo apt install openssh-client
    # In both: Lb01, Lb02
    sudo apt install rsync
- To balance the load between nodes:
    ```bash
    # In lb01 (Master)
    sudo apt install haproxy
    ```
- To control the VIP address:
    ```bash
    # Both here: Lb01, Lb02
    sudo apt install keepalived -y
    ```
- To check the status between nodes:
    ```bash
    # Both here: Lb01, Lb02
    sudo apt install conntrackd -y
    ```
- To set programmed tasks:
    - crontab
- As every web development you need a firewall, **IN ALL NODES**:
    ```bash
    sudo apt install ufw -y
    ```
---

## 4. Example of Web Developing:
I'm going to set a easy html and php coding to see web in production, **HERE YOU CAN SET YOUR OWN CODE**:
**In Lb01 and Lb02**:
```html
<h1>Welcome to my high availability web service</h1>
<footer>
    Served by: <?php echo gethostname(); ?> <br>
    IP: <?php echo $_SERVER['SERVER_ADDR']; ?>
</footer>
```

Now you have to restart the server to set the HTML you set before to production webpage:
```bash
sudo sytemctl restart apache2
# It's always recommended then see if it's all OK:
sudo systemctl status apache
```
It has to be:
![image.png](images/image.png)

---

## 5. Data Sync between nodes:
First we must have a **Key Pair** to login Master to Backup server to sync data:
So, in **Web01** we will generate them:
```bash
ssh-keygen -t ed25519
# The following options don't matter, so you can press enter
```
Now we have to send it to the server:
```bash
ssh-copy-id Web02@192.168.1.22
# Put the Lb02 password and you'll have them in your server in /home/username/.ssh/authorizedkeys
```

Now in **Web01** we will develop an script to sync it automatically every day 2 time per day.
```bash
#!/bin/bash

SOURCE=/var/www/html/documentname.php
DESTINATION="Web02@192.168.1.22:/var/www/html/documentname.php"
LOGFILE="/var/log/rsync_sync.log"

echo "--- Sync started at $(date) ---" >> $LOGFILE
rsync -avz $SOURCE $DESTINATION >> $LOGFILE 2>&1

if [ $? -eq 0 ]; then
    echo "Sync successful." >> $LOGFILE
else
    echo "Sync FAILED. Check network or SSH-keys." >> $LOGFILE
fi
```

Then you have to give execution permission:
```bash
chmod +x script.sh
```

Now we will set to execute 2 times per day:

Press **crontab -e** and add this at the end of the file:
```text
0 */12 * * * /bin/bash /home/username/script.sh
```

---

## 6. Implementing Load Balancer:
I'm going to use HAproxy in **Lb01 (Master**) to do the balancing between nodes.

- Configuration file (**/etc/haproxy/haproxy.cfg**), (**Put this at the end of the file**):
    ```text
    frontend http_front
    bind *:80
    default_backend web_backend
    backend web_backend
    balance roundrobin
    server web01 192.168.1.21:80 check
    server web02 192.168.1.22:80 check
    ```
- To see your interface you have to do:
    ```bash
    ip a
    ```
    Your IP is the one in front of your IP
- Once you have done this, you must configure **VIP** address (**/etc/keepalived/keepalived.conf**):
    ```text
    vrrp_instance VI_1 {
        state MASTER
        interface YOURINTERFACE
        virtual_router_id 51
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            192.168.1.100
        }
    }
    ```
- Reload and verity the status:
    ```bash
    sudo systemctl restart keepalived && sudo systemctl status keepalived
    ```

- Once we finished the **Master**, now configure the **Slave** or **Backup**:
    ```text
    vrrp_instance VI_1 {
        state SLAVE
        interface YOURINTERFACE
        virtual_router_id 51
        priority 90
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            192.168.1.100
        }
    }
    ```
- Restart and see the status.

---

## 7. Syncing / Checking status between nodes:
- **conntrackd** is used the check every few seconds the status between nodes and if one fails it send the status to the other.

- Once installed in [step 2](#2-tools-we-are-going-to-use) you must add this at the end of the file (**/etc/conntrackd/conntrackd.conf**):

    ```text
    Sync {
        Mode FTFW {
            ResendQueueSize 1024
            ACKWindowSize 300
        }
        Multicast {
            IPv4_address 225.0.0.50
            Group 3780
            Interface YOURINTERFACE
        }
    }
    ```

- As always reload and check the correct status.

---

## 8. Security and Firewall:
Important to set right ports to entry connections in every server:

- In Web Cluster (Web01, Web02):
    ```bash
    sudo ufw allow 80/tcp
    # We are going to active ssh connections but ONLY from our network
    sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

    # Enabling the firewall
    sudo ufw enable
    ```

- In the Load Balancers / Firewalls (Lb01, Lb02):
    ```bash
    sudo ufw allow 80/tcp
    # We are going to active ssh connections but ONLY from our network
    sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

    # Enabling the firewall
    sudo ufw enable
    ```

- We have to allow the **VRRP** (VIP address) and **conntrackd** (checking status) connections:
    ```bash
    # In Lb01:
    sudo ufw allow from 192.168.1.11 to any port 3780 proto udp
    ```

    ```bash
    # In Lb02:
    sudo ufw allow from 192.168.1.10 to any port 3780 proto udp
    ```

- Finally reload the firewall:
    ```bash
    sudo ufw reload
    ```

---

## 9. Proof of Concept:
First we going to access the entry portal in ***192.168.1.100***:
![image2](images/image2.png)

Testing High Availability:
- Now we are goint to do infinite *ping* to test the availability and shutdown the ***Master*** server, it should output one failed ping but the VIP should move to ***Slave*** one:
![images3](images/image3.png)

- Checking that the slave has de .100 IP:
![image4](images/image4.png)

- Verify the output logs in */var/log/conntrackd.log*:
![image5](images/image5.png)

- Testing *failover* between web nodes:
    - Here I will shutdown Web01 and try if Web02 responds:
    ![images6](images/image6.png)

    It responds through the same IP and if we check the text we know that is the Server 2 responding (*192.168.1.22*)

---

## 10. Analyzing Improvements and Point of Failure:
- If Web01 fails before it sync the data to Web02 all changes we made  on this time will be lost. **It should be every time code is updated.**
- HAproxy configurations is only made at Lb01, if this Load Balancer fails, it will fail all the server because is the *single point of failure*.
- UFW is configured only through one direction, now both, it will not update the rules if the Lb01 makes the failover, this should be bidirectional.

---

## 11. Who am I?
- Currently I am studying cybersecurity and doing an Internship in a Company working with Amazon Web Services.
- In my free time I like to improve my ethical hacking knowledge, programming open-source tools, you can see them in my profile. IT is an amazing world that more things you know you realize you don't know nothing because you are always learning and improving yourself.
- It's my second Proof Of Concept but it's not going to be the last one I'm pretty sure about that.

---

## 12. Greetings to:
- To my teachers, they teach me how IT works and research always for new information and update me every day because IT never ends.
- To my girlfriend to support me with everything and trying to understand this kind of things that to normal people are very cumbersome.
- And obvious to my parents to support me with studies and all.
- You for reading this PoC if you had arrive to this part ahaha.

---

<h1 style="text-align: center;">By v0id0100</h1>