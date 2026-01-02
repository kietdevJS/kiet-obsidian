- In parallel vm setting, select shared network (NAT)
	![[Screenshot 2025-11-02 at 19.03.26 1.png]]
- Network preferences: get the subnet address
	![[Screenshot 2025-11-02 at 19.04.18.png]]

- In the vm , sudo /etc/netplan/... to set static ip:
	![[Screenshot 2025-11-02 at 19.05.30.png]]
- To allow other machines communicate each other in a local network, set hostname by sudo vi etc/hosts
	![[Screenshot 2025-11-02 at 19.07.27.png]]
- sudo /etc/hostname to modify machine hostname
- sudo netplan apply  