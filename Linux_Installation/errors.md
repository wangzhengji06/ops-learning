# Cannot ping from host to 10.0.0.13

Reason: Wrongly added bad route

Solution: `ip route`  `sudo ip route del 10.0.0.0/24 via 10.0.0.2 dev ens33`

# Cannot ssh as root

Reason: Password not set / Ubuntu does not allow root ssh on default

Solution:
1. `sudo passwd root` 
2. `vim /etc/ssh/sshd_config` and add `PermitRootLogin yes`
3. `sudo systemctl restart ssh`

