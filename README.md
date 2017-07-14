# Fail2ban configuration

We're running fail2ban as basic protection against brute force ssh
attempts. In the default configuration 4 failed attempts within 10
minutes will result in a 10 minute ban. There is also a manual blacklist
intented to catch people who are repeatedly unbanned. It lives at
/etc/fail2ban/ip.blacklist and it is just a list of IP addresses. 

## Using this blacklist

I use this blacklist by cloning out the repository into
/etc/fail2ban/fail2ban-permanent, then symlinking the jail, action and filter
files
```
  $ cd /etc/fail2ban/jail.d
  $ ln -s ../fail2ban-permanent/jail.d/blacklist.conf
  $ cd ../action.d
  $ ln -s ../fail2ban-permanent/action.d/blacklist.conf
  $ cd ../filter.d
  $ ln -s ../fail2ban-permanent/filter.d/blacklist.conf

  $ systemctl restart fail2ban # Or service on CentOS 6 machines
```

## Unbanning an IP

To unban an IP, start by removing it from the blacklist
```
  sed -i '/<IP ADDRESS>/d' /etc/fail2ban/ip.blacklist
```
Then unban it fromt he current ruleset
```
  fail2ban-client set blacklist-auto unbanip <IP ADDRESS>
```

You can verify the IP has been unbanned by examining iptables (`iptables -nL`).


## Blacklist operation

There is a new jail (jail.d/blacklist.conf) which monitors the blacklist file
for modifications. It maintains an iptables chain called f2b-blacklist and
should set the contents of the blacklist on startup. The bantime is set to -1 so
these entries will not expire. The findtime should be set to something very long
(currently a year), with the intention that it covers all of the relevant
timestamps in the ip.blacklist file. Items older than this in the ip.blacklist
should periodically be removed. I *think* banned items will still live in the
fail2ban database `/var/lib/fail2ban/fail2ban.sqlite3`, but I'm not sure about
that!

Good candidates for blacklisting can be found by grepping the log files of
fail2ban. I like to watch the Unban action and if an IP is being repeatedly
banned and unbanned I'll throw it in the blacklist. Depending on where fail2ban
is logging and your ssh jail name (ssh-iptables), the following command pipe
would show you IP addresses banned more than 10 times.
```
grep -E '[ssh-iptables\]\s+Unban' /var/log/fail2ban.log \
  | awk '{print $NF}' \
  | sort -n \
  | uniq -c \
  | sort -n \
  | awk '$1 > 10 {print $2}' \
  | sort -n
```

A shellscript is included in this repository to implement something like the
above and to make updating the blacklist a bit more pleasant. In its default
operating mode, findBruteSSH will look at /var/log/fail2ban and scrape out IP
addresses with more than 10 Unbans. If you use the -u flag, these will be
compared with existing entries in ip.blacklist and a list of updates will
created. This update list is in the right format for adding to the blacklist
```
  ./findBruteSSH -u >> ip.blacklist
```
The format for the ip.blacklist file is 
```
2017-07-13 22:15:56 Ban 10.100.100.1
# i.e. $(date +'%Y-%m-%d %T') Ban <ip>
```
The blacklist jail should notice the changes and add the new IPs to the iptables
f2b-blacklist chain. There is a `-n <NUM>` flag if you would like to change the
threshold for unbans, and a `-f <LOGFILE>` option if you would like to examine a
different log file.

Because the blacklist is configured as a jail rather than a startup item the
usual fail2ban-client commands can be used for banning, unbanning and status,
e.g.
```
  fail2ban-client status blacklist
Status for the jail: blacklist
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	254
|  `- File list:	/etc/fail2ban/ip.blacklist
`- Actions
   |- Currently banned:	1
   |- Total banned:	1
   `- Banned IP list:	10.100.100.1
```


## BUGS
The first time it is used fail2ban might not load your ip.blacklist file. I
think this might be the notification backend, but just manually adding one more
ban item to the list should trigger the update.
```
  $ echo "$(date +'%Y-%m-%d %T' Ban <IP>)" >> /etc/fail2ban/ip.blacklist
```
In general if you are trying to debug fail2ban, it is worth destroying the
database (it'll be rebuild on startup)
```
  $ rm /var/lib/fail2ban/fail2ban.sqlite3
  $ systemctl restart fail2ban
```
And the fail2ban-regex utility is useful for checking your filter syntax
```
  fail2ban-regex /etc/fail2ban/ip.blacklist /etc/fail2ban/filter.d/blacklist.conf 
```

## Using blacklist on other machines

There are a couple of places where the configuration is specific to my machines,
but they should not be hard to find and fix, in particular.

 * The jail name (e.g ssh-iptables) is hard coded inside findBruteSSH's regex
 * The Unban action regex has changed a few times (e.g. between CentOS 6 & 7.
   Again, this is hard coded in findBruteSSH
 * The filter.d/blacklist.conf filters should be checked with fail2ban-regex
 * The blacklist file location is hard coded in places
