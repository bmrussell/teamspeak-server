Rules in

/etc/iptables/rules.v4
/etc/iptables/rules.v6

sudo iptables-save > /etc/iptables.rules
sudo iptables-restore < /etc/iptables.rules
sudo netfilter-persistent save

sudo ip6tables-save > /etc/ip6tables.rules
sudo ip6tables-restore < /etc/ip6tables.rules
sudo netfilter-persistent save