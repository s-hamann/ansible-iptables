iptables
========

Configures the firewall using iptables.
This role supports ingress and egress filtering.
When egress filtering is enabled, some common protocols are automatically allowed.
This includes:
* DNS requests to the name servers in `/etc/resolv.conf` or by systemd-resolved
* Package management (Gentoo, Debian, Alpine)

Requirements
------------

Requires a kernel built with iptables support.

On Debian and when `iptables_output_policy` is set to `DROP`, `dig` from `dnsutils` is required so that certain mirrors (e.g. `deb.debian.org`) are whitelisted correctly.

Role Variables
--------------

`iptables_input_policy`  
Default policy for `INPUT` chain if nothing else matches
Valid values are `ACCEPT` (default) and `DROP`.

`iptables_output_policy`  
Default policy for `OUTPUT` chain if nothing else matches
Valid values are `ACCEPT` (default) and `DROP`.

`iptables_log_dropped_input`, `iptables_log_dropped_output`  
When set to `true`, dropped packages on the `INPUT`/`OUTPUT` chain are logged.
Logging is limited to 1 packet per second.

`iptables_allow_ping`  
Whether to allow incoming *and* outgoing ICMP and ICMPv6 echo requests.
Valid values are `true` (default) and `false`.

`iptables_allow_broadcast`  
Whether to allow incoming IPv4 broadcast traffic.
Valid values are `true` (default) and `false`.

`iptables_allow_multicast`  
Whether to allow incoming IPv4 multicast traffic.
Valid values are `true` (default) and `false`.

`iptables_allow_anycast`  
Whether to allow incoming IPv4 anycast traffic.
Valid values are `true` (default) and `false`.

`iptables_accept_ip_input`, `iptables_block_ip_input`, `iptables_accept_ip_output`, `iptables_block_ip_output`  
Each of these variables contains a list of IP addresses (or hostnames) for which all traffic should be accepted/blocked in the `INPUT`/`OUTPUT` chain.
To invert the match, IP addresses (and hostnames) can be prefixed with `!`.
The rules are set up early in the respective chains, which means that they take precedence over almost all other rules.

`iptables_accept_tcp_input`, `iptables_block_tcp_input`, `iptables_accept_udp_input`, `iptables_block_udp_input`, `iptables_accept_tcp_output`, `iptables_block_tcp_output`, `iptables_accept_udp_output`, `iptables_block_udp_output`  
These variables can be used to set up port-specific rules that allow/block traffic to a certain TCP/UDP port on the `INPUT`/`OUTPUT` chain, respectively.
Accept rules take precedence over block rules.

All of these variables are lists.
The list entries can simply be port numbers, which sets up a rule for these ports without any further conditions.
It is also possible to create more elaborate rules by setting a list entry to a dictionary.
In this case, the dictionary must to contain the port number in the key `port`.
A number of optional keys can be added:
* `sport`: The source port to match on.
* `source`: The source address to match on.
* `destination`: The destination address to match on.
* `user`: The name of the local user that the traffic originates from. Only works on the `OUTPUT` chain.

Ports, addresses and users can be prefixed with `!` to invert the match.

Example:
```yaml
iptables_accept_tcp_input:
  - 22
  - 23
  - { port: 80, source: 10.0.0.0/8 }
  - { port: 80, source: 192.168.0.0/16 }
  - { port: 443, source: !172.16.0.0/12 }
iptables_block_udp_output:
  - { port: 53, user: root }
  - 123
```

`iptables_allow_dhcp4`, `iptables_allow_dhcp6`  
If set to `true`, open the ports required for network configuration via DHCPv4 or DHCPv6, respectively, on all interfaces.
Defaults to `true`.

`iptables_dns_resolver_user`  
If the output policy is `DROP`, setting this allows this local user account to do arbitrary DNS requests.
This is useful when running a local DNS resolver, such as Unbound or dnsmasq.
The default is empty, i.e. only traffic to the configured DNS server in `/etc/resolv.conf` and by systemd-resolved is allowed.

`iptables_ntp_client_user`  
If the output policy is `DROP`, setting this allows this local user account to do arbitrary NTP requests.
Default is empty, i.e. no rules for NTP are created.

`iptables_smtp_client_user`  
If the output policy is `DROP`, setting this allows this local user account to connect to arbitrary hosts via SMTP.
This is useful when running a local MTA, such as Postfix or Exim.
Default is empty, i.e. no rules for SMTP are created.

`iptables_unrestricted_users`, `iptables_unrestricted_groups`  
When `iptables_output_policy` is set to `DROP`, these variables can be used to allow specific users (mostly) unrestricted access to the network.
Note that address and port specific rules have a higher precedence, so listing users here may not grant totally unrestricted access.
When whitelisting groups, iptables rules for the individual users are created.
When group memberships change, the firewall script needs to be restarted to apply the changes to the rules.

`iptables_custom_input_rules`, `iptables_custom_output_rules`  
The previous setting allow setting up commonly used rules without dealing with the complexity of iptables.
Unfortunately, this abstraction also takes away some of the power of iptables.
These variables allow writing raw iptables rules and therefore provide more control.

A few things are to note:
* The rules are added directly to the `INPUT` or `OUTPUT` chain, respectively.
* The rules are added at the end of these chains, which means that the rules generated by other settings take precedence over these here.
* The rules are automatically applied to both `iptables` and `ip6tables`.
  Rules that start with `-4` or `-6` are added to `iptables` or `ip6tables` only, respectively.
* Instead of `-j DROP` or `-j REJECT` it is preferable to use `-j drop_input` or `-j drop_output`.
  This ensures consistent behaviour and also handle logging (if enabled).

Example:
```yaml
iptables_custom_input_rules:
  - '-p tcp --dport 22 --comment "just for the sake of the example" -j ACCEPT'
  - '-p tcp --dport 23 --comment "just for the sake of the example" -j drop_input'
```

### Managing rules using groups

This role uses several list variables (e.g. `iptables_accept_tcp_input`) to configure the firewall.
It may make sense to define them in groups.
However, when using the variable name directly, Ansible does not merge the lists.
Instead, the last defined variable overwrites earlier definitions.

To facilitate group-driven firewall configuration and avoid duplication of data, this role allows appending an arbitrary suffix to each list's variable name to make it unique.
These lists are then merged into one.

For example, the group `ssh-servers` may define:
```yaml
iptables_accept_tcp_input_ssh:
  - 22
```
And the group `web-servers` may define:
```yaml
iptables_accept_tcp_input_web:
  - 80
  - 443
```
For a system in both groups, this would result in:
```yaml
iptables_accept_tcp_input:
  - 22
  - 80
  - 443
```
Lists defined this way can be overwritten in sub-groups, if desired.
For example, a group `secure-web-servers`, which is a child of `web-servers`, could redefined the list:
```yaml
iptables_accept_tcp_input_web:
  - 443
```
A system in this group would then not accept connection on TCP port 80.

Notes:
* Suffixes can be completely arbitrary but an `_` needs to be between the original list name and the suffix.
* The resulting lists are lexicographically ordered by the suffix name.
* If the "base" variable (without a suffix) is defined, the respective suffixed variables are ignored.

License
-------

MIT
