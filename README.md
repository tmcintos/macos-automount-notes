# automounting NFS shares under `/Network` with modern macOS

## The Problem

We want to automount several NFS shares from a server on our local network at `/Network/{Applications,Library,Users,share,usrlocal}` in the Mac file system.  Furthermore, we want this to work with a MacBook running macOS 13.1, regardless of whether it is on the local network or not.

## Assumptions

* We have at least two Network Locations:
	1. `Automatic` (left at macOS defaults)
	2. `Home`
* With the `Home` Network Location, we've arranged (e.g. using local DNS) that:
	1. the MacBook's host name is `macbook.example.net`,
	2. the NFS server is reachable at `nfs.example.net`,
	3. the NFS server is also reachable at `tbt-nfs.example.net` if directly connected to the MacBook via Thunderbolt networking connection. *This secondary address is optional, used to obtain the highest speed connection possible when docked at the desk where the NFS server is located.*
* With any other Network Location, the Macbook has a different host name.

## Preliminaries

With SIP enabled, modern macOS security policy ensures that:

1. **`automountd` cannot mount directly on `/Network`**. This rules out the possibility of describing `/Network` using an indirect automount map, or similar. So instead we'll use a direct map separately describing each path under `/Network`.
2. **`automountd` cannot mount directly on `/Network/Library`**. To work around this, we'll have the automounter mount on `/Network/.Library` instead, then provide a symbolic link from `/Network/Library` to `/Network/.Library`.

To set this up:

1. Reboot into Recovery Mode
2. `cd /Volumes/Data`
3. `mkdir -p Network/.Library`
4. `ln -s .Library Network/Library`

Create `private/etc/synthetic.conf` containing the line:

	Network		System/Volumes/Data/Network

and then reboot.

*Note that `automountd` would normally create the synthetic link [6] from `/Network` to `/System/Volumes/Data/Network` as required on modern macOS with ROSV (*Read-only System Volume*), but creating the `Network` directory manually as needed to provide the `Library` symlink prevents this from happening, so we create it manually.*

## First Attempt

Starting with the macOS default `/etc/auto_master` [2]:

	#
	# Automounter master map
	#
	+auto_master		# Use directory service
	#/net				-hosts		-nobrowse,hidefromfinder,nosuid
	/home				auto_home	-nobrowse,hidefromfinder
	/Network/Servers	-fstab
	/-					-static

We create a new automount direct map [1] file `/etc/auto_Network`:

	/System/Volumes/Data/Network/Applications	tbt-nfs,nfs:/Exports/Local/MacOSX/Applications
	/System/Volumes/Data/Network/.Library		tbt-nfs,nfs:/Exports/Local/MacOSX/Library
	/System/Volumes/Data/Network/Users			tbt-nfs,nfs:/Exports/Users
	/System/Volumes/Data/Network/usrlocal		tbt-nfs,nfs:/Exports/Local/MacOSX/usrlocal
	/System/Volumes/Data/Network/share			tbt-nfs,nfs:/Exports/Local/share

(*with client-side NFS failover configured to try `tbt-nfs.example.net` first, then `nfs.example.net`*).

We then add the following line to end of `/etc/auto_master` to enable this automount map:

	/-					auto_Network	-soft,intr,retrycnt=0,retrans=1,dumbtimer,timeo=1,deadtimeout=1

## Let's use Open Directory Instead

Alternatively, we can create `auto_master` and `auto_Network` maps in Open Directory (LDAP) instead of making the file modifications above. This way we don't have to repeat the configuration if we need to support mulitple NFS client machines using the same Network account server:

To do this, we create `~/Documents/automount.ldif` (*assuming Open Directory server is `ldap.example.net` at `192.168.1.1`; replace `dc=ldap,dc=example,dc=net` and `192.168.1.1` below with the appropriate values for your server*) with the following contents:

	# auto_master
	dn: automountMapName=auto_master,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: top
	objectClass: automountMap
	automountMapName: auto_master
	description: Must be initialized from ~/Documents/automount.ldif using ldapadd(1)
	
	# auto_master: /- auto_Network
	dn: automountKey=/-,automountMapName=auto_master,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: automount
	automountInformation: auto_Network -bg,soft,intr,retrycnt=0,retrans=1,dumbtimer,timeo=1,deadtimeout=1
	automountKey: /-
	description: auto_master entry for auto_Network map
	
	# auto_Network
	dn: automountMapName=auto_Network,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: top
	objectClass: automountMap
	automountMapName: auto_Network
	description: auto_Network map
	
	# auto_Network: /System/Volumes/Data/Network/Applications	tbt-nfs,nfs:/Exports/Local/MacOSX/Applications
	dn: automountKey=/System/Volumes/Data/Network/Applications,automountMapName=auto_Network,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: automount
	automountInformation: tbt-nfs,nfs:/Exports/Local/MacOSX/Applications
	automountKey: /System/Volumes/Data/Network/Applications
	description: Applications entry of auto_Network map
	
	# auto_Network: /System/Volumes/Data/Network/Users			tbt-nfs,nfs:/Exports/Users
	dn: automountKey=/System/Volumes/Data/Network/.Library,automountMapName=auto_Network,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: automount
	automountInformation: tbt-nfs,nfs:/Exports/Local/MacOSX/Library
	automountKey: /System/Volumes/Data/Network/.Library
	description: .Library entry of auto_Network map
	
	# auto_Network: /System/Volumes/Data/Network/Users			tbt-nfs,nfs:/Exports/Users
	dn: automountKey=/System/Volumes/Data/Network/Users,automountMapName=auto_Network,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: automount
	automountInformation: tbt-nfs,nfs:/Exports/Users
	automountKey: /System/Volumes/Data/Network/Users
	description: Users entry of auto_Network map
	
	# auto_Network: /System/Volumes/Data/Network/usrlocal		tbt-nfs,nfs:/Exports/Local/MacOSX/usrlocal
	dn: automountKey=/System/Volumes/Data/Network/usrlocal,automountMapName=auto_Network,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: automount
	automountInformation: tbt-nfs,nfs:/Exports/Local/MacOSX/usrlocal
	automountKey: /System/Volumes/Data/Network/usrlocal
	description: usrlocal entry of auto_Network map
	
	# auto_Network: /System/Volumes/Data/Network/share			tbt-nfs,nfs:/Exports/Local/share
	dn: automountKey=/System/Volumes/Data/Network/share,automountMapName=auto_Network,cn=automountMap,dc=ldap,dc=example,dc=net
	objectClass: automount
	automountInformation: tbt-nfs,nfs:/Exports/Local/share
	automountKey: /System/Volumes/Data/Network/share
	description: share entry of auto_Network map

We then (re)load this into LDAP:

	ldapdelete -r -h 192.168.1.1 -U diradmin -W -v 'automountMapName=auto_master,cn=automountMap,dc=ldap,dc=example,dc=net' 'automountMapName=auto_Network,cn=automountMap,dc=ldap,dc=example,dc=net'
	ldapadd -h 192.168.1.1 -U diradmin -W -v -f ~/Documents/automount.ldif

See also: `ldif(5)` and <https://superuser.com/questions/621147/unable-to-add-automount-entries-on-macosx-using-directory-utility>

## What About Network Locations?

The MacBook is a mobile device. With the configuration above, when we're not connected to the local network (where our NFS server is unavailable), we'll have problems resulting from automount RPC timeouts (*e.g. when passing references to `/Network/Library` are made during login*). This mostly stems from a [hardcoded RPC timeout in automountd](https://github.com/apple-oss-distributions/autofs/blob/autofs-306/automountd/autod_nfs.c#L129).

As long as we can guarantee a unique hostname will be used when connected to the local network, and that the NFS server on the local network remains generally available, we can mitigate this problem by

1. using a hostname-specific direct automount map;
2. invoking `automount -vc` to reload the automountd configuration whenever the Network Location is changed.

### LDAP-based approach

Starting with the LDAP-based configuration above, we could make the following changes to `/etc/auto_master`:

1. comment out `+auto_master` (don't include `auto_master` map from Open Directory)
2. add static map `auto_$HOST`

Here's the new `/etc/auto_master`:

	#
	# Automounter master map
	#
	#+auto_master		# Use directory service
	#/net				-hosts		-nobrowse,hidefromfinder,nosuid
	#/home				auto_home	-nobrowse,hidefromfinder
	/Network/Servers	-fstab
	/-					-static
	/-					auto_$HOST	-soft,intr,retrycnt=0,retrans=1,dumbtimer,timeo=1,deadtimeout=1

We then create `/etc/auto_macbook.example.net` containing one line:

	+auto_Network		# include auto_Network map from directory service

### Non-LDAP-based approach

Alternatively, if not using LDAP, we could simply rename the `/etc/auto_Network` file previously described above to `/etc/auto_macbook.example.net`.

## Making it Automatic

We want to run `automount -vc` whenever the Network Location is changed, so that this works as intended.  We can do that with a `launchd.plist(5)`, as follows:

Create `/Library/LaunchDaemons/net.example.automounts.plist` (*replacing `net.example` with your own prefix below*):

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
	<plist version="1.0">
	<dict>
		<key>Label</key>
		<string>net.example.automounts</string>
		<key>ProgramArguments</key>
		<array>
			<string>/usr/sbin/automount</string>
			<string>-vc</string>
		</array>
		<key>WatchPaths</key>
		<array>
			<string>/Library/Preferences/SystemConfiguration/preferences.plist</string>
			<string>/etc/auto_master</string>
		</array>
		<key>RunAtLoad</key>
		<true/>
	</dict>
	</plist>
	
Then load it:
	
	sudo launchctl load /Library/LaunchDaemons/net.example.automounts.plist

## References:

1. [Autofs: Automatically Mounting Network File Shares in Mac OS X (Autofs.pdf)](https://web.archive.org/web/20151212185550/http://www.apple.com/business/docs/Autofs.pdf)
2. [`auto_master(5)`](https://www.freebsd.org/cgi/man.cgi?query=auto_master&sektion=5#MAP_SYNTAX)
3. `automount(8)`
4. `automountd(8)`
5. `autofs.conf(5)`
6. `synthetic.conf(5)`

## See Also

* [Autofs on Mac OS X](https://gist.github.com/rudelm/7bcc905ab748ab9879ea)
