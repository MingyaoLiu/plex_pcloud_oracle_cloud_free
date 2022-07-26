# plex_pcloud_oracle_cloud_free

Documenting complete steps for personal reference in the future. Free tier allowance might change in the future, all of the free tier quota is as of July 2022, if you see this guide in the future, the quota may have been different. Oracle cloud does give you 300 bucks for 30 day trial, so even if you make a mistake on the free quota, it won't be the end of the day.

Oracle Cloud:

1. Sign up for oracle cloud, choose a region (cannot change in the future I believe)
2. create a compute instance
  a. Choose ubuntu (plex support for arm64 is limited to debian as of July 2022, don't feel like compiling myself)
  b. Choose Ampere A1 (4 OCPU core, 24 GB ram)
  c. Update boot block device to 200GB, slide VPU to max 120 for peak IOPS: 45000 IOPS and Throughput: 360 MB/s
  d. make sure generate a public ip, default networking should be ok, then create the instance.
4. Update network ingress security to allow plex port ingress, so it is easier to connect to it after install plexmediaserver.
  a. go to oracle cloud -> networking -> virtual cloud networks -> subnet details -> Default Security List for subnet, click add Ingress Rules, add the following (the udp rule is probably useless, but I added it anyway):
  
SSH:

1. convert pem to ppk with puttygen, and use it to connect ssh to the server. put `ubuntu@xxx.xxx.xxx.xxx` so it automatically login with ubuntu user.
2. To allow plex port access, in addition to the security rule on Oracle Cloud, we also need to add a rule in the iftable
  a. update iptable to allow 32400 port access: (for tcp) `sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 32400 -j ACCEPT` (for udp) `sudo iptables -I INPUT 6 -m state --state NEW -p udp --dport 32400 -j ACCEPT`
  b. save the iftable configuration so we don't need to update it everytime after reboot: `sudo netfilter-persistent save`



