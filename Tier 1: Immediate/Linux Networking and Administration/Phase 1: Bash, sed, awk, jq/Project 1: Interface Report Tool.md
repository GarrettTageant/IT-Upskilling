# Claude-generated project description

## Project 1: interface report tool. Write ifreport.sh [-j] [-i IFACE]. It reports, for each interface, the name, operational state, MAC address, IPv4 addresses, and receive and transmit byte counters.

Take state and addresses from ip -j addr and process them with jq.
Take counters from /proc/net/dev and process them with awk.
With -j, emit valid JSON built with jq, not string concatenation.
Add a separate sed command that replaces the address of one named host in a copy of /etc/hosts and keeps a backup.

## Pass criteria

shellcheck reports no findings.
The script exits with status 2 for an invalid option and status 1 for a nonexistent interface.
-j output passes jq empty.
Counters in two consecutive runs do not decrease.
The sed command changes only the line for the named host.
