Client is able to ping IP addresses (e.g. 8.8.8.8, 1.1.1.1, etc.), but not *domain names*.

Results in basically **no internet connection**.
MASQUERADE rules are set up in iptables,

```
<SNIP>
POSTROUTING
MASQUERADE  all  --  10.8.0.0/24          anywhere
<SNIP>
```

dhcp-options are set,
```
# Routing
push "redirect-gateway def1"
push "dhcp-option DNS 1.1.1.2"
```
(i have even tried multiple dhcp-option DNSes)

net.ipv4.ip_forward is set in `/etc/sysctl.confl`
`net.ipv4.ip_forward=1`

I have also configured DNS settings on the client (Android) as well: 
1. Turning off *Private DNS*; 
2. Changing to a custom DNS;
3. Overriding the VPN server's DNS; and
4. Changing clients;
**and nothing has worked so far.**
