# plex_pcloud_oracle_cloud_free

Documenting complete steps for personal reference in the future. Free tier allowance might change in the future, all of the free tier quota is as of July 2022, if you see this guide in the future, the quota may have been different. Oracle cloud does give you 300 bucks for 30 day trial, so even if you make a mistake on the free quota, it won't be the end of the day.


## Setup Oracle Cloud

1. Sign up for oracle cloud, choose a region (cannot change in the future I believe)

2. create a compute instance

3. Choose ubuntu (plex support for arm64 is limited to debian as of July 2022, don't feel like compiling myself)
  
4. Choose Ampere A1 (4 OCPU core, 24 GB ram)
  
5. Update boot block device to 200GB, slide VPU to max 120 for peak IOPS: 45000 IOPS and Throughput: 360 MB/s
  
6. make sure generate a public ip, default networking should be ok, then create the instance.


## Set up SSH

1. convert pem to ppk with puttygen, and use it to connect ssh to the server. put `ubuntu@xxx.xxx.xxx.xxx` so it automatically login with ubuntu user.

2. go to `SSH -> Auth`, and add the private ppk.

3. Give it a name and save this configuration


## Allow Plex Port 32400 on Oracle Cloud Instance

1. go to `oracle cloud -> networking -> virtual cloud networks -> subnet details -> Default Security List for subnet`, click add Ingress Rules, add the following (the udp rule is probably useless, but I added it anyway):

![ingress setting](https://github.com/MingyaoLiu/plex_pcloud_oracle_cloud_free/blob/main/oracle_ingress_rules.png?raw=true)
  
2. In addition to the security rule on Oracle Cloud, we also need to add a rule in the instance's iftable

3. SSH into the instance, update iptable to allow 32400 port access: 
  
      (for tcp) `sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 32400 -j ACCEPT` 
  
      (for udp) `sudo iptables -I INPUT 6 -m state --state NEW -p udp --dport 32400 -j ACCEPT`
  
  
4. save the iftable configuration so we don't need to update it everytime after reboot: `sudo netfilter-persistent save`
  
 
## Install PlexMediaServer and update User

1. download plexmedia server with `sudo wget https://downloads.plex...xxxxx..._arm64.deb`
  
2. install with `sudo dpkg -i plexmediaserver...xxx..._arm64.deb`
  
3. However since plexmediaserver is running with non-login user plex, it doesn't have enough file permission for our pcloud mounted files, we will change the runnign user of plex to ubuntu, so that it 100% sure have the file permission:
  
4. stop plex service: `sudo systemctl stop plexmediaserver`
  
5. add override: `sudo systemctl edit plexmediaserver`, and add:
    
   ```
   [Service]
   User=ubuntu
   Group=ubuntu
   ```
      
   to the space above `### Lines below this comment will be discarded`
      
6. reload daemon `sudo systemctl daemon-reload`
    
7. change file ownership of plex media server files: `sudo chown -R ubuntu:ubuntu /var/lib/plexmediaserver`
    
8. start plexmediaserver again: `sudo systemctl start plexmediaserver` or reboot the server
    
9. Now you should be able to access the server from the external ip of the oracle cloud compute instance. However to set up a new plex server, we need local access first, and to do this we need to tunnel the 32400 port to local PC with ssh. Add tunnel in `PuTTY -> SSH -> Tunnels`

![PuTTY Tunnels](https://github.com/MingyaoLiu/plex_pcloud_oracle_cloud_free/blob/main/Putty_tunnel.png?raw=true)
  
10. Now you can access the 32400 port from `http://localhost:32400/web`, finish the setup process for plex without adding any library.
  
11. Make sure you remove the Tunnel in PuTTY, otherwise you may not be able to access plex when using SSH.
  
 
## Build and Install pCloud console client:

1. Follow the compile instruction here: https://github.com/pcloudcom/console-client
  
2. There might be missing build required package, see here: https://github.com/pcloudcom/console-client/pull/158
  
3. If enabled 2FA on pCloud, may need to use a fetched version of console-client. See here: https://github.com/pcloudcom/console-client/pull/163
  
4. Make sure you to the mount point directory manually under user `ubuntu` with `mkdir /home/ubuntu/pcloud_data`, if the directory doesn't exist, pcloud might fail or stuck.
  
5. (example) using the 2FA PR: `pcloudcc -u [MY@EMAIL.COM] -p -t [2FA_CODE] -r -m /home/ubuntu/pcloud_data -s`
  
6. If successful, you should see log similar to this: 
  
  ```
  pCloud console client v.2.0.1
  Please, enter password
  Down: Everything Downloaded| Up: Everything Uploaded, status is LOGIN_REQUIRED
  logging in
  Down: Everything Downloaded| Up: Everything Uploaded, status is CONNECTING
  Down: Everything Downloaded| Up: Everything Uploaded, status is TFA_REQUIRED
  Down: Everything Downloaded| Up: Everything Uploaded, status is CONNECTING
  Down: Everything Downloaded| Up: Everything Uploaded, status is SCANNING
  eventxxxxxxxxxx
  eventxxxxxxxxxx
  Down: Everything Downloaded| Up: Everything Uploaded, status is READY
  ```
  
7. Doesn't need to use pcloudcc daemon -d, will create a system service in the next step so it starts after system boot.
  
  
## Make pCloud a system service that start with the system

1. Create a script `sudo nano /home/ubuntu/start_pcloud.sh`:
  
  ```
  #! /bin/bash
  /usr/local/bin/pcloudcc -u [MY@EMAIL.COM] -m /home/ubuntu/pcloud_data
  ```
    
2. Make this script executable: `sudo chmod +x start_pcloud.sh`
    
3. Create a service file `sudo nano /etc/systemd/system/pcloud.service`:
  
  ```
  [Unit]
  Description=pCloud Start Service
  After=network.target
  StartLimitBurst=5
  StartLimitIntervalSec=10

  [Service]
  User=ubuntu
  Group=ubuntu
  Type=simple
  ExecStart=/home/ubuntu/start_pcloud.sh
  Restart=on-failure
  RestartSec=1

  [Install]
  WantedBy=multi-user.target
  ```
  
4. Start the pcloud service: `sudo systemctl start pcloud`
  
5. if mounted pcloud directory works and you can see all the files, enable pcloud service to startup: `sudo systemctl enable pcloud`
  
6. reboot your server, and everything should work correctly.
