# Lsyncd Training Session

### Manifest

- Readme.md - This file. Duh.
- environment - Vagrant environment for servers

# Lsyncd Technical Session

## So what is lsyncd?

Lsyncd is a tool used to keep a source directory in sync with other local or remote directories.  It is a solution suited keeping directories in sync by batch processing changes over to the synced directories.¬

## When would we use lsyncd?

So the generic use case is to keep a source directory in sync with one or more local and remote directories.

This could mean:

-  Creating a live backup of a directory which would be easy to fail over to.
-  Eliminate a single point of failure by distributing the data to multiple servers
-  Scale out a web application (e.g. Wordpress) 

## How does lsyncd work?

#### Rsync + SSH

Primarily lsyncd is used as a combination of rsync + ssh. This is how we keep folders on remote servers in sync with our source.

Lsyncd is written in the Lua language and thus the configuration is valid Lua syntax. This allows us to configure lsyncd at various depths of complexity. On their website the configuration is broken down in to four layers. You can read further on this topic in their [documentation](https://axkibe.github.io/lsyncd/manual/config/file/).

####  Configuration

Lsyncd's configuration file is located at _/etc/lsyncd.conf_. In here we find how to configure the behavior of lsyncd. The two main sections for our configuration file are *settings* and *sync*.

In the settings section we define some of the global options for our daemon.

```
settings {
        logfile         = "/var/log/lsyncd/lsyncd.log",
        statusFile      = "/var/log/lsyncd/lsyncd.stat",
        statusIntervall = 1,
        nodaemon        = false
}
```

> **Caution**: You may see the settings directive started off as "_settings = {...}_". In previous versions the configuration file *settings* was defined as a variable. It is now a function and thus no longer needs an "=" between settings and {

Here is an example of a sync section.

```
sync {
        default.rsyncssh,
        source = "/var/www/html",   
        targetdir = "/var/www/html",
        host = "192.168.33.20",
        delay           = 5,
        rsync = { rsh="/usr/bin/ssh -l webuser -i /home/webuser/.ssh/id_rsa -o StrictHostKeyChecking=no"},
        ssh = {
       		_extra = {'-l','webuser','-i','/home/webuser/.ssh/id_rsa','-o','StrictHostKeyChecking=no'}
       	}
}
```


It is possible to use just rsync or bash to keep two locations in sync locally. We do not cover these in this session and instead of focus mainly on using rsync + ssh.

#### Batch Processing

Lsyncd monitors files and directories for changes. These changes are observed, aggregated and batched out to the target servers. The **default interval for batching is 15 seconds**.  This can be modified by either the *maxDelays* in the *settings* section or the Delay directive in the *sync* section.

> **Note on File Monitoring**: This is done via the file monitoring interface ( inotify or fsevents ). The max number of monitors is defined under  */proc/sys/fs/inotify/max_user_watches*

**Example** Setting maxDelays configures the number of events to queue up before running an rysnc.
```
settings {
...
maxDelays = 10
...
}
```

**Example:** Delay here sets how long between syncing the queued events.
```
sync {
	default.rsyncssh,
	...,
	Delay = 5,
	...
}
```

## Setup and Installation

As of this writing (8/29/2016) the most recent version 	available in the EPEL channel is lsyncd-2.1.5. This section will cover installation and setup on a pair of CentOS 6 servers.

#### Prerequisites

- Two or more servers.
- The appropriate EPEL channel configured
- lsyncd from the EPEL channel
- SSH Keys
- A source and target location defined

#### Installation

For our CentOS 6 servers, installation is pretty straight forward.

```
# yum -y install lsyncd
```

#### SSH Keys

Let us assume we have two servers, web1 and web2. 

- Web1 - Our source server has the IP of 192.168.33.10
- Web2 - Our target server has the IP of 192.168.33.20

In this example we are creating ssh-keys on web1 and distributing them to web2.

```
# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
a7:a3:70:ce:f7:1c:11:a4:56:cd:7a:47:87:c3:1b:a3 root@lb
The key's randomart image is:
+--[ RSA 2048]----+
|          oo . . |
|         +  o B .|
|        o .. o * |
|       .  ..E o  |
|        S o. .   |
|         o .     |
|    . . o .      |
|     = ..o .     |
|      +. .o      |
+-----------------+
# ssh-copy-id user@web2
```

#### Source and Target Locations

Here is an example configuration file. This could be copied and pasted to have a working configuration. We will discuss it in detail below.

```
-- Two dashes define a comment
settings {
        logfile         = "/var/log/lsyncd/lsyncd.log",
        statusFile      = "/var/log/lsyncd/lsyncd.stat",
        statusIntervall = 1,
        nodaemon        = false,
}

sync {
        default.rsyncssh,
        source = "/var/www/html",
        host = "192.168.33.20",
        targetdir = "/var/www/html",
        delay           = 5,
        rsync = { rsh="/usr/bin/ssh -l webuser -i /home/webuser/.ssh/id_rsa -o StrictHostKeyChecking=no"},
        ssh = {
       		_extra = {'-l','webuser','-i','/home/webuser/.ssh/id_rsa','-o','StrictHostKeyChecking=no'}
       	}
}
```

Under the *settings* function, most items are self explanatory, but let us talk about *statusInterval* and "nodaemon". 

> **statusInterval = 1** -   writes the status file at shortest after this number of seconds has passed (default: 10) 

>  **nodaemon = false** - Determines if lsyncd runs as  daemon or in the foreground. 

With the sync section there is a bit more to elaborate on.

> **default.rsyncssh** - Defines we are using SSH in addition to rsync to sync remote hosts.
> **source = "/var/www/html"**  - The path we want to sync from
> **host = "192.168.33.20"** - The host we want to sync to
> **targetdir = "/var/www/html"** - The destination path we want to sync to
>  **delay = 5**  - Changing the default batch time from 15 to 5 seconds.
>  **rsync = {rsh="/usr/bin/ssh -l webuser -i /home/webuser/.ssh/id_rsa -o StrictHostKeyChecking=no"}** - This section provides a way to pass along additional rsync options. In our example we are connecting as the webuser user.
>  **ssh = {_extra = {'-l','webuser','-i','/home/webuser/.ssh/id_rsa','-o','StrictHostKeyChecking=no'}}**  - This section is needed because lsyncd executes moves and deletes by issuing a ssh command instead of a rsync command.

#### One extra Note

> The **-o StrictHostKeyChecking=no** setting is not 100% needed but it will make life easier if you are syncing with a server you have never logged in to. It by passes having to ssh in to a server and accept the host key before syncing.


#### And That's It

With our servers setup, the keys in place and the lsyncd.conf file setup we should be good to start the service up and see our files start to sync up.

```
# chkconfig lsyncd on
# service lsyncd start
```
Upon start up we can look at our log and stat file to gather some information.

*/var/log/lsyncd/lsyncd.log*
```
Tue Aug 30 04:57:54 2016 Normal: recursive startup rsync: /var/www/html/ -> 192.168.33.20:/var/www/html/
Tue Aug 30 04:57:55 2016 Normal: Startup of "/var/www/html/" finished: 0
```

/var/log/lsyncd/lsyncd.start

```
Lsyncd status report at Tue Aug 30 04:58:05 2016

Sync1 source=/var/www/html/
There are 0 delays
Excluding:
  nothing.


Inotify watching 136 directories
  1: /var/www/html/
  2: /var/www/html/wordpress/
  3: /var/www/html/wordpress/wp-content/
  4: /var/www/html/wordpress/wp-content/plugins/
  5: /var/www/html/wordpress/wp-content/plugins/akismet/
  6: /var/www/html/wordpress/wp-content/plugins/akismet/_inc/
```

## Possible Issues with File Syncing

#### WARNING - Restarting lsyncd can delete files on remote servers and you may not be expecting it to. - WARNING

From lsyncd documentation.

>By default Lsyncd will delete files on the target that are not present at the source since this is a fundamental part of the idea of keeping the target in sync with the source. However, many users requested exceptions for this, for various reasons, so all default implementations take delete as an additional parameter.

The default the *delete* directive is true. Which is Lsyncd will delete on the target whatever is not in the source. At startup and what's being deleted during normal operation. 

Other Valid Options

> **delete 	= 	false** 	Lsyncd will not delete any files on the target. Not on startup nor on normal operation. (Overwrites are possible though)

> **delete 	= 	'startup'** 	Lsyncd will delete files on the target when it starts up but not on normal operation.

> **delete 	= 	'running'** Lsyncd will not delete files on the target when it starts up but will delete those that are removed during normal operation

So by default lsyncd will delete any files on the targets not currently on the source. This keeps with the idea of keeping one path in sync with a source path.

If you find yourself in a place where data is being written to one of the target directories outside of the lsyncd process, restarting lsyncd on the master **WILL DELETE THOSE FILES.**.

If you find yourself in this situation, before restarting lsyncd you will want to try and sync all of the targets back to the source and then restart.

#### SSH Keys Permission Issues

If you follow the configuration file step by step you will see the following SSH line as part of the configuration. This line passes extra parameters allowing us to move and delete files as webuser. 

    ssh = {_extra = {‘-l’,’webuser’,’-i’,’/home/webuser/.ssh/id_rsa’,’-o’,’StrictHostKeyChecking=no’}} 

Without this configuration in place there will be an error similar to what you see below.

```
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
Tue Sep  6 19:29:03 2016 Normal: Retrying (list): 255
Tue Sep  6 19:29:08 2016 Normal: Deleting list
```

Having both the rsync and the ssh configuration allows all operations to be executed as webuser.
