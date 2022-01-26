# mod_evasive

Fork of mod_evasive for Apache 2.4. Original module by Deep Logic, Inc

WHAT IS MOD_EVASIVE ?

mod_evasive is an evasive maneuvers module for Apache to provide evasive action in the event of an HTTP DoS or DDoS attack or brute force attack. It is also designed to be a detection tool, and can be easily configured to talk to ipchains, firewalls, routers, and etcetera.

Detection is performed by creating an internal dynamic hash table of IP Addresses and URIs, and denying any single IP address from any of the following:

Requesting the same page more than a few times per second
Making more than 50 concurrent requests on the same child per second
Making any requests while temporarily blacklisted (on a blocking list)
This method has worked well in both single-server script attacks as well as distributed attacks, but just like other evasive tools, is only as useful to the point of bandwidth and processor consumption (e.g. the amount of bandwidth and processor required to receive/process/respond to invalid requests), which is why it's a good idea to integrate this with your firewalls and routers.

This module instantiates for each listener individually, and therefore has a built-in cleanup mechanism and scaling capabilities. Because of this, legitimate requests are rarely ever compromised, only legitimate attacks. Even a user repeatedly clicking on 'reload' should not be affected unless they do it maliciously.

Three different module sources have been provided:

Apache v1.3 API: mod_evasive.c Apache v2.0 API: mod_evasive20.c Apache v2.4 API: mod_evasive24.c NSAPI (iPlanet): mod_evasiveNSAPI.c *

## HOW TO WORKS

A web hit request comes in. The following steps take place:

The IP address of the requestor is looked up on the temporary blacklist
The IP address of the requestor and the URI are both hashed into a "key". A lookup is performed in the listener's internal hash table to determine if the same host has requested this page more than once within the past 1 second.
The IP address of the requestor is hashed into a "key". A lookup is performed in the listerner's internal hash table to determine if the same host has requested more than 50 objects within the past second (from the same child).
If any of the above are true, a 403 response is sent. This conserves bandwidth and system resources in the event of a DoS attack. Additionally, a system command and/or an email notification can also be triggered to block all the originating addresses of a DDoS attack.

Once a single 403 incident occurs, mod_evasive now blocks the entire IP address for a period of 10 seconds (configurable). If the host requests a page within this period, it is forced to wait even longer. Since this is triggered from requesting the same URL multiple times per second, this again does not affect legitimate users.

The blacklist can/should be configured to talk to your network's firewalls and/or routers to push the attack out to the front lines, but this is not required.

mod_evasive also performs syslog reporting using daemon.alert. Messages will look like this:

Jan 19 17:59:33 <HOSTNAME>  mod_evasive[22349]: Blacklisting address X.X.X.X: possible DoS attack.

## WHAT IS THIS TOOL USEFUL FOR?

This tool is excellent at fending off request-based DoS attacks or scripted attacks, and brute force attacks. When integrated with firewalls or IP filters, mod_evasive can stand up to even large attacks. Its features will prevent you from wasting bandwidth or having a few thousand CGI scripts running as a result of an attack.

If you do not have an infrastructure capable of fending off any other types of DoS attacks, chances are this tool will only help you to the point of your total bandwidth or server capacity for sending 403's. Without a solid infrastructure and address filtering tool in place, a heavy distributed DoS will most likely still take you offline.

## HOW TO INSTALL

APACHE V2.4
-----------
1. Install pre-requisite packages (httpd-devel / gcc)

2. Extract this archive

3. Run apxs -i -a -c mod_evasive24.c

This step, The module will be built and installed into $APACHE_ROOT/modules, and loaded into your httpd.conf
For apxs - APache eXtenSion tool

## About the options:

-i: This indicates the installation operation and installs one or more dynamically shared objects into the server's modules directory.

-a This activates the module by automatically adding a corresponding LoadModule line to Apache's httpd.conf configuration file, or by enabling it if it already exists.

-c: This indicates the compilation operation. It first compiles the C source files (.c) of files into corresponding object files (.o) and then builds a dynamically shared object in dsofile by linking these object files plus the remaining object files (.o and .a) of files. If no -o option is specified the output file is guessed from the first filename in files and thus usually defaults to mod_name.so.

refLink: https://httpd.apache.org/docs/2.4/programs/apxs.html

NOTE: If you want add load module differently in step 3, in separate file:

3. execute: apxs -i -c mod_evasive24.c and include follow lines in conf.modules.d directory. i.e: 

```
echo 'LoadModule evasive24_module /usr/lib64/httpd/modules/mod_evasive24.so' > /etc/httpd/conf.modules.d/00-evasive.conf
```
4. Restart Apache

5. Add mod_evasive.conf:
E.g.: vim /etc/httpd/conf.d/mod_evasive.conf

<IfModule mod_evasive24.c>
    DOSHashTableSize    3097
    DOSPageCount        2
    DOSSiteCount        50
    DOSPageInterval     1
    DOSSiteInterval     1
    DOSBlockingPeriod   10
</IfModule>

Optionally you can also add the following directives:

    DOSEmailNotify	you@yourdomain.com
    DOSSystemCommand	"su - someuser -c '/sbin/... %s ...'"
    DOSLogDir		"/var/lock/mod_evasive"

DOSHashTableSize
----------------

The hash table size defines the number of top-level nodes for each child's
hash table.  Increasing this number will provide faster performance by
decreasing the number of iterations required to get to the record, but
consume more memory for table space.  You should increase this if you have
a busy web server.  The value you specify will automatically be tiered up to
the next prime number in the primes list (see mod_evasive.c for a list
of primes used).

DOSPageCount
------------

This is the threshhold for the number of requests for the same page (or URI)
per page interval.  Once the threshhold for that interval has been exceeded,
the IP address of the client will be added to the blocking list.

DOSSiteCount
------------

This is the threshhold for the total number of requests for any object by
the same client on the same listener per site interval.  Once the threshhold
for that interval has been exceeded, the IP address of the client will be added
to the blocking list.

DOSPageInterval
---------------

The interval for the page count threshhold; defaults to 1 second intervals.

DOSSiteInterval
---------------

The interval for the site count threshhold; defaults to 1 second intervals.

DOSBlockingPeriod
-----------------

The blocking period is the amount of time (in seconds) that a client will be
blocked for if they are added to the blocking list.  During this time, all
subsequent requests from the client will result in a 403 (Forbidden) and
the timer being reset (e.g. another 10 seconds).  Since the timer is reset
for every subsequent request, it is not necessary to have a long blocking
period; in the event of a DoS attack, this timer will keep getting reset.

DOSEmailNotify
--------------

If this value is set, an email will be sent to the address specified
whenever an IP address becomes blacklisted.  A locking mechanism using /tmp
prevents continuous emails from being sent.

NOTE: Be sure MAILER is set correctly in mod_evasive.c
      (or mod_evasive20.c).  The default is "/bin/mail -t %s" where %s is
      used to denote the destination email address set in the configuration.
      If you are running on linux or some other operating system with a
      different type of mailer, you'll need to change this.

DOSSystemCommand
----------------

If this value is set, the system command specified will be executed
whenever an IP address becomes blacklisted.  This is designed to enable
system calls to ip filter or other tools.  A locking mechanism using /tmp
prevents continuous system calls.  Use %s to denote the IP address of the
blacklisted IP.

DOSLogDir
---------

Choose an alternative temp directory

By default "/tmp" will be used for locking mechanism, which opens some
security issues if your system is open to shell users.

  	http://security.lss.hr/index.php?page=details&ID=LSS-2005-01-01

In the event you have nonprivileged shell users, you'll want to create a
directory writable only to the user Apache is running as (usually root),
then set this in your httpd.conf.


httpd.conf or apache2.conf
--------------------------

LoadModule evasive24_module modules/mod_evasive24.so

(This line is already added to your configuration by apxs)

WHITELISTING IP ADDRESSES

IP addresses of trusted clients can be whitelisted to insure they are never
denied.  The purpose of whitelisting is to protect software, scripts, local
searchbots, or other automated tools from being denied for requesting large
amounts of data from the server.  Whitelisting should *not* be used to add
customer lists or anything of the sort, as this will open the server to abuse.
This module is very difficult to trigger without performing some type of
malicious attack, and for that reason it is more appropriate to allow the
module to decide on its own whether or not an individual customer should be
blocked.

To whitelist an address (or range) add an entry to the Apache configuration
in the following fashion:

DOSWhitelist	127.0.0.1
DOSWhitelist	127.0.0.*

Wildcards can be used on up to the last 3 octets if necessary.  Multiple
DOSWhitelist commands may be used in the configuration.

TWEAKING APACHE

The keep-alive settings for your children should be reasonable enough to
keep each child up long enough to resist a DOS attack (or at least part of
one).  Remember, it is the child processes that maintain their own internal
IP address tables, and so when one exits, so does all of the IP information it
had. For every child that exits, another 5-10 copies of the page may get
through before putting the attacker back into '403 Land'.  With this said,
you should have a very high MaxRequestsPerChild, but not unlimited as this
will prevent cleanup.

You'll want to have a MaxRequestsPerChild set to a non-zero value, as
DosEvasive cleans up its internal hashes only on exit.  The default
MaxRequestsPerChild is usually 10000.  This should suffice in only allowing
a few requests per 10000 per child through in the event of an attack (although
if you use DOSSystemCommand to firewall the IP address, a hole will no
longer be open in between child cycles).

TESTING

Want to make sure it's working? Run test.pl, and view the response codes.
It's best to run it several times on the same machine as the web server until
you get 403 Forbidden messages. Some larger servers with high child counts
may require more of a beating than smaller servers before blacklisting
addresses.

If you use AllowOverride None(Default) and dont't have any content in root context dir, change for All and use test.html in your <DOCUMENT_ROOT> dir, i.e:
```
sed -i 's/AllowOverride None/AllowOverride All/g' /etc/httpd/conf/httpd.conf
sed -i 's/DirectoryIndex index.html/DirectoryIndex index.html test.html/g' /etc/httpd/conf/httpd.conf
cat << EOF > /var/www/html/test.html
<!DOCTYPE html>
<html>
    <head>
        <title>Test Page</title>
    </head>
    <body>
        <p>This is an example of a simple HTML page with one paragraph.</p>
    </body>
</html>
EOF

# restart apache

# Execute test
perl test.pl

# Expected Output example
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
HTTP/1.1 403 Forbidden
```


Please don't use this script to DoS others without their permission.

KNOWN BUGS

- This module appears to conflict with the Microsoft Frontpage Extensions.
  Frontpage sucks anyway, so if you're using Frontpage I assume you're asking
  for problems, and not really interested in conserving server resources anyway.
