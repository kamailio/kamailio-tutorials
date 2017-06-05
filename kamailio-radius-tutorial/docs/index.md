# Kamailio With RADIUS Backend

**Authors**

  * Daniel-Constantin Mierla
    * miconda [at] gmail.com
  * Elena-Ramona Modroiu

## Abstract ##

Step by step tutorial about starting a basic VoIP service using Kamailio as SIP
server (softswitch) and FreeRadius server as AAA server (backend).

**Important Note**: the tutorial was written to be used with Kamailio
(OpenSER) v1.0.1 and FreeRadius v1.1.0 on a Debian unstable (sid) system. It
can be used with other Linux/Unix distributions if you know the proper
replacements for Debian specific tools (e.g., apt, GNU make).

## Overview ##

[Kamailio (former OpenSER)](https://www.kamailio.org) is a scalable and
flexible SIP (RFC3261) server with a lot of features that can power your VoIP
services. Features like ENUM lookup, TLS support, authentication,
authorization and accounting against database or Radius servers, least cost
routing, load balancing or Call Processing Language allow providing
residential or carrier VoIP services.

[FreeRADIUS](http://www.freeradius.org) is an open source RADIUS server. It
scales from embedded systems with small amounts of memory, to systems with
millions of users. It is fast, flexible, configurable, and supports many
authentication servers.

This document focuses on configuring FreeRadius to offer AAA services to
Kamailio SIP server, needed for VoIP services. Step by step configuration is
provided and you must be logged in as 'root' user to be able to execute the
commands exampled in this document. It is assumed that you have basic knowledge
about Linux or Debian operation and administration.

## The Architecture Of The VoIP Service ##

Communication between VoIP users (phones) and the server uses the SIP (RFC3261)
protocol. The SIP server will communicate with the AAA server via radius client,
which is linked to the SIP server. Users' profile can be stored to the local
file system (as it is done is the examples included in this document) or other
storage system supported by AAA server (e.g., database, ldap).

```
    +---------+                                              +---------+
    | PHONE 1 |<--SIP--+    +-------------------+            | STORAGE |
    +---------+        |    |    SIP SERVER     |            +----+----+
                       #===>|    (Kamailio)     |                 |        
    ...........        |    |-------------------|         +-------+-------+
                       |    |   RADIUS CLIENT   |<==AAA==>| RADIUS SERVER |
    +---------+        |    | (radiusclient-ng) |         |  (FreeRADIUS) |
    | PHONE N |<--SIP--+    +-------------------+         +---------------+
    +---------+
```

## FreeRadius Installation ##

FreeRadius server is part of official Debian distribution. To install it just
do:

```
  ~# apt-get install freeradius
```

If you install FreeRadius from Debian packages, the configuration files are
located in `/etc/freeradius/`.

If you prefer to install FreeRadius from sources, then go to
[FreeRADIUS web site](http://www.freeradius.org) and download it from there.
Please note that next commands are the basic steps to install FreeRadius from
source - you should read FreeRadius installation documentation to do it
properly.

```
  ~# mkdir -p /usr/local/src/freeradius
  ~# cd /usr/local/src/freeradius
  ~# wget ftp://ftp.freeradius.org/pub/radius/freeradius-1.1.0.tar.gz
  ~# tar xvfz freeradius-1.1.0.tar.gz
  ~# cd freeradius-1.1.0
  ~# ./configure
  ~# make
  ~# make install
```

If you install FreeRadius from sources, the configuration files are located in
`/usr/local/raddb/`.

## Kamailio Installation ##

To compile the Radius support in OpenSER you must install first the `radiusclient-ng` library.

### RadiusClient-ng Library Installation ##

The homepage for `radiusclient-ng` library is:

  * http://developer.berlios.de/projects/radiusclient-ng/

You can get Debian packages for `radiusclient-ng` from Kamailio download site:

```
~# mkdir -p /usr/local/src/radiusclient-ng
~# cd /usr/local/src/radiusclient-ng
~# wget \
  http://kamailio.org/pub/openser/1.0.1/packages/deb-sid/libradius-ng-dev_0.5.1_i386.deb
~# wget \
  http://kamailio.org/pub/openser/1.0.1/packages/deb-sid/libradius-ng_0.5.1_i386.deb
~# wget \
  http://kamailio.org/pub/openser/1.0.1/packages/deb-sid/radiusclient-ng_0.5.1_i386.deb
~# dpkg -i libradius-ng libradius-ng-dev radiusclient-ng
```

The configuration files are placed in: `/etc/radiusclient-ng/`.

If you want to install `radiusclient-ng` library from sources, go to:

  * http://developer.berlios.de/projects/radiusclient-ng/

and download the tarball with sources in `/usr/local/src/radiusclient-ng`.

```
  ~# cd /usr/local/src/radiusclient-ng
  ~# tar xvfz radiusclient-ng-X.Y.Z.tar.gz
  ~# cd radiusclient-ng-X.Y.Z
  ~# ./configure
  ~# make
  ~# make install
```

If you have installed from sources, the configuration files are placed in:
`/usr/local/etc/radiusclient-ng/`.

### Kamailio Installation From Sources ###

Since `acc` module requires compilation by hand to enable Radius accounting, in
this document will be detailed only installation of Kamailio with Radius support
from sources.

```
  ~# mkdir -p /usr/local/src/kamailio
  ~# cd /usr/local/src/kamailio
  ~# wget https://kamailio.org/pub/kamailio/1.0.1/src/kamailio-1.0.1_src.tar.gz
  ~# tar xvfz kamailio-1.0.1_src.tar.gz
  ~# cd kamailio-1.0.1
```

To enable Radius accounting, edit the `modules/acc/Makefile` and uncomment the
part related to Radius accounting. You can comment the part related to SQL
(database) accounting.

Next, edit `Makefile` and remove from `exclude_modules` all modules that have
`_radius` in their name. You can remove from `exclude_modules` the `mysql`
module as well -- the configuration file for Kamailio presented in this document
uses it.

To keep persistent user location you need to install MySQL server, client, and development library (more details at http://www.mysql.com).

```
  ~# apt-get install mysql-server mysql-client libmysqlclient-dev
```

Once you have accomplished the above steps you can proceed with compilation and
installation of Kamailio.

```
  ~# cd /usr/local/src/openser/kamailio-1.0.1
  ~# NICER=1 make all
  ~# make install
```

To create the MySQL database required by Kamailio, use:

```
  ~# DBENGINE=MYSQL kamdbctl create
```

### Kamailio RADIUS Dictionary ###

Kamailio comes with a RADIUS dictionary which is required to interwork with
FreeRADUIS server. By default, the Kamailio RADIUS dictionary is installed in:

  * `/usr/local/etc/openser/dictionary.radius`

The dictionary presented in the next example is a bit tuned to include
translation for SIP method's names and to work with the configuration file of
Kamailio which is included in this document.

```
#
# SIP RADIUS attributes
#
# Schulzrinne indicates attributes according to
# draft-schulzrinne-sipping-radius-accounting-00
#
# Sterman indicates attributes according to
# draft-sterman-aaa-sip-00
#
# Proprietary indicates an attribute that hasn't
# been standardized
#
# Check out http://www.iana.org/assignments/radius-types
# for up-to-date list of standard RADIUS attributes
# and values
#

#
# NOTE: All standard (IANA registered) attributes are 
#       commented out except those that are missing in 
#       the default dictionary of the radiusclient-ng 
#       library.
#


#### Attributes ###
#ATTRIBUTE User-Name		         1  string     # RFC2865
#ATTRIBUTE Service-Type		         6  integer    # RFC2865
#ATTRIBUTE Called-Station-Id             30  string     # RFC2865, acc
#ATTRIBUTE Calling-Station-Id            31  string     # RFC2865, acc
#ATTRIBUTE Acct-Status-Type              40  integer    # RFC2865, acc
#ATTRIBUTE Acct-Session-Id               44  string     # RFC2865, acc
ATTRIBUTE Sip-Method                   101  integer    # Schulzrinne, acc
ATTRIBUTE Sip-Response-Code            102  integer    # Schulzrinne, acc
ATTRIBUTE Sip-Cseq                     103  string     # Schulzrinne, acc
ATTRIBUTE Sip-To-Tag                   104  string     # Schulzrinne, acc
ATTRIBUTE Sip-From-Tag                 105  string     # Schulzrinne, acc
ATTRIBUTE Sip-Translated-Request-URI   107  string     # Proprietary, acc
ATTRIBUTE Sip-Src-IP                   108  string     # Proprietary, acc
ATTRIBUTE Sip-Src-Port                 109  string     # Proprietary, acc
ATTRIBUTE Digest-Response      206  string     # Sterman, auth_radius
ATTRIBUTE Sip-Uri-User         208  string     # Proprietary, auth_radius
ATTRIBUTE Sip-Group            211  string     # Proprietary, group_radius
ATTRIBUTE Sip-Rpid             213  string     # Proprietary, auth_radius
ATTRIBUTE SIP-AVP              225  string     # Proprietary, avp_radius
ATTRIBUTE Digest-Realm                1063  string     # Sterman, auth_radius
ATTRIBUTE Digest-Nonce                1064  string     # Sterman, auth_radius
ATTRIBUTE Digest-Method               1065  string     # Sterman, auth_radius
ATTRIBUTE Digest-URI                  1066  string     # Sterman, auth_radius
ATTRIBUTE Digest-QOP                  1067  string     # Sterman, auth_radius
ATTRIBUTE Digest-Algorithm            1068  string     # Sterman, auth_radius
ATTRIBUTE Digest-Body-Digest          1069  string     # Sterman, auth_radius
ATTRIBUTE Digest-CNonce               1070  string     # Sterman, auth_radius
ATTRIBUTE Digest-Nonce-Count          1071  string     # Sterman, auth_radius
ATTRIBUTE Digest-User-Name            1072  string     # Sterman, auth_radius

### CISCO Vendor Specific Attributes ###
#VENDOR Cisco              9
#ATTRIBUTE Cisco-AVPair    1   string   Cisco           # VSA, auth_radius

### Acct-Status-Type Values ###
#VALUE Acct-Status-Type     Start             1         # RFC2866, acc
#VALUE Acct-Status-Type     Stop              2         # RFC2866, acc
VALUE Acct-Status-Type     Failed           15         # RFC2866, acc

### Service-Type Values ###
VALUE Service-Type      Call-Check       10   # RFC2865, uri_radius
VALUE Service-Type      Group-Check      12   # Proprietary, group_radius
VALUE Service-Type      Sip-Session      15   # Schulzrinne, acc, auth_radius
VALUE Service-Type      SIP-Caller-AVPs  30   # Proprietary, avp_radius
VALUE Service-Type      SIP-Callee-AVPs  31   # Proprietary, avp_radius

VALUE Sip-Method        INVITE            1         # Proprietary, acc
VALUE Sip-Method        CANCEL            2         # Proprietary, acc
VALUE Sip-Method        ACK               4         # Proprietary, acc
VALUE Sip-Method        BYE               8         # Proprietary, acc

```

You can either paste the example above in a new file `/etc/radiusclient-ng/dictionary.kamailio`, or copy
`/usr/local/etc/openser/dictionary.radius` to
`/etc/radiusclient-ng/dictionary.kamailio`.

```
  cp /usr/local/etc/openser/dictionary.radius \
    /etc/radiusclient-ng/dictionary.kamailio
```

## FreeRadius Configuration ##

This part refers only to the configuration items strict related to to the components that interact with `radiusclient-ng` library and `Kamailio` server.

**Note**: the files to whom we refer below are located either in
`/etc/freeradius` or `/usr/local/etc/raddb`.

### Clients configuration ###

FreeRadius server allows connections only from RADIUS clients which know a
shared secret and connect from a certain IP.

The list with allowed clients is kept in the file `clients.conf`. You should
add there a new entry having the following format:

```
client kamailio.host.ip {
	secret		= shared-secret
	shortname	= kamailio.host
}
```

For example, if Kamailio is running on `10.10.10.10`, you should add in
`clients.conf`:

```
client 10.10.10.10 {
	secret		= testing123
	shortname	= kamailio
}
```

### Main Configuration File ###

The main configuration file for FreeRadius is `radiusd.conf`. You must load
the `digest` module. In the “modules” section you should have:

```
	modules {
		...
		#
		#  The 'digest' module currently has no configuration.
		#
		#  "Digest" authentication against a Cisco SIP server.
		#  See 'doc/rfc/draft-sterman-aaa-sip-00.txt' for details
		#  on performing digest authentication for Cisco SIP servers.
		#
		digest {
		}
		...
	}
```

Also, you must enable `digest` authentication and authorization. You must
uncomment `digest` lines in `authenticate` and `authorize` sections.

```
	authorize {
		...
		#
		#  If you have a Cisco SIP server authenticating against
		#  FreeRADIUS, uncomment the following line, and the 'digest'
		#  line in the 'authenticate' section.
		digest
		...
	}
	...
	authenticate {
		...
		#
		#  If you have a Cisco SIP server authenticating against
		#  FreeRADIUS, uncomment the following line, and the 'digest'
		#  line in the 'authorize' section.
		digest
		...
	}
```

### FreeRADIUS Dictionary ###

You must include Kamailio RADIUS dictionary file in FreeRADIUS dictionary.
FreeRADIUS main dictionary file is placed in `/etc/freeradius/dictionary` or
`/usr/local/etc/raddb/dictionary`. You have to edit it and add the next line.

```
	$INCLUDE /etc/radiusclient-ng/dictionary.kamailio
```

### FreeRADIUS Users ###

By default the user profiles (username, domain, password) are stored in
`/etc/freeradius/users` or `/usr/local/etc/raddb/users`. This is a text file
and you can edit easily. Alternatively, the user profiles can be loaded from
a database, for this, please read the documentation of `FreeRADIUS`.

Next is presented an example of what you have to insert in `users` file. The
example includes records for AVPs, group membership checking and user
authentication. These examples are the records necessary to run the `OpenSER`
config presented below.

```
### --- avps ---
101@kamailio.org Auth-Type := Accept, Service-Type == "SIP-Callee-AVPs"
	Sip-Avp += "#3#1",
	Sip-Avp += "#4:08:00",
	Sip-Avp += "#5:16:00",
	Sip-Avp += "#6:Mon,Wed,Thu,Fri"

102@kamailio.org Auth-Type := Accept, Service-Type == "SIP-Callee-AVPs"
	Sip-Avp += "#3#1",
	Sip-Avp += "#4:08:00",
	Sip-Avp += "#5:16:00",
	Sip-Avp += "#6:Mon,Wed,Thu,Free"

DEFAULT Auth-Type := Accept, Service-Type == "SIP-Callee-AVPs"

### --- group checking ---
### --- user 101 ---
101@kamailio.org Auth-Type := Accept, Sip-Group == "voip", Service-Type == "Group-Check"
	Reply-Message = "Authorized"

101@kamailio.org Auth-Type := Accept, Sip-Group == "pstn", Service-Type == "Group-Check"
	Reply-Message = "Authorized"

### --- user 102 ---
102@kamailio.org Auth-Type := Accept, Sip-Group == "voip", Service-Type == "Group-Check"
	Reply-Message = "Authorized"

DEFAULT Auth-Type := Reject, Service-Type == "Group-Check"

### --- user authentication ---
101@kamailio.org Auth-Type := Digest, User-Password == "101"
	Reply-Message = "Authenticated",
	Sip-Avp += "rpid:101",
	Sip-Avp += "#2:192.168.2.10",
	Sip-Avp += "#2:192.168.2.11"

102@kamailio.org Auth-Type := Digest, User-Password == "102"
	Reply-Message = "Authenticated",
	Sip-Avp += "rpid:102",
	Sip-Avp += "#2:192.168.2.12"
```

## RadiusClient-ng Configuration ##

The radiusclient-ng library has to be configured to connect and authenticate
to the RADIUS server. Also, the dictionary of the library has to include the
attributes required by Kamailio.

### Main configuration file ###

The main configuration file is `radiusclient.conf` (located either in
`/usr/local/etc/radiusclient-ng/` or `/etc/radiusclient-ng/`). In this file
you have to set the address for authentication and accounting servers.

Locate the lines presented in the next image and change the addresses to the
appropriate values.

```
authserver      localhost
...
acctserver      localhost
```

You have to change `localhost` with the address of your RADIUS server. If
RADIUS server and Kamailio run on the same system, then you can leave them
unchanged. For example, assuming that RADIUS server is running on
`10.10.10.11`, the above lines must be changed in:

```
authserver      10.10.10.11
...
acctserver      10.10.10.11
```

`authserver` specifies the authentication and authorization server.

`acctserver` specifies the accounting server.

### Servers Configuration ###

For each server you set in `radiusclient.conf` you have to specify the secret
to be used when communicating with it. The file where you have to write the
secret is `servers`, in the same folder as `radiusclient.conf`. The format of
the file is text and contains server address and associated secret, on the
same line, separated by whitespaces.

Make sure that the secret is the same with the one set in `clients.conf` of
RADIUS server.

```
10.10.10.11	testing123
```

### Dictionary File ###

The dictionary file of radiusclient-ng library must include the attributes for Kamailio. Edit the `dictionary` file located either in
`/usr/local/etc/radiusclient-ng/` or `/etc/radiusclient-ng/` and add the
following line:

```
  $INCLUDE /etc/radiusclient-ng/dictionary.openser
```

## Testing RADIUS Server ##

If you want to test the RADIUS server, independent of Kamailio usage, follow
the next steps.

Add a test user in `users` file of FreeRADIUS.

```
test Auth-Type := Digest, User-Password == "test"
	Reply-Message = "Hello, test with digest"
```

Start RADIUS server in debug mode.

```
freeradius -X
# or
radiusd -X
```

Create a file named `digest` and put following in it, all in a single line:

```
User-Name = "test", Digest-Response = "631d6d73147add2f9e437f59bbc3aeb7", 
Digest-Realm = "testrealm", Digest-Nonce = "1234abcd" , 
Digest-Method = "INVITE", Digest-URI = "sip:5555551212@example.com", 
Digest-Algorithm = "MD5", Digest-User-Name = "test"
```

Use `radclient` for testing the server. It is assumed that you run `radclient`
on Kamailio system. You have to install it there, since this tool comes with
FreeRADIUS server.

```
radclient -f digest 10.10.10.11 auth testing123
```

In case of correct response from the server, you should see something like:

```
Received response ID 224, code 2, length = 45
        Reply-Message = "Hello, test with digest"
```

## Kamailio Configuration ##

The next example present a configuration file of Kamailio using most of the
operations that can be done with RADIUS.

In the scenario implemented, the server authenticates the user based on password
and source IP. To each user profile is associated a list of IP addresses from
where the user can log in. As you can see in 'FreeRADIUS Users' section, the AVP
having the ID 2 stores the IP addresses, those AVPs being loaded upon digest
authentication.

Furthermore, to show the usage of `avp_load_radius()`, the config includes a
time-based blocking of incoming calls. The users store as `callee` avps the
details required to check the time of allowed incoming calls. In the AVP with
ID 3 is stored a flag that enables/disables the time-based checking. AVP with
ID 4 stores the staring hour and AVP with ID 5 the end hour. In the AVP with
ID 6 is stored the list of days within the week when such calls are allowed.

Looking at user `101`, he will receive calls only Monday, Wednesday, Thursday
and Friday, between 8:00AM and 4:00PM.

Extra RADIUS accounting is exampled via source IP address and port. To compare
the results, the accounting information is written to syslog as well.

Group membership checking is performed to allow access to different types of
services. There are used the following groups:

  * `suspended` - if the user belongs to this group, he is not allowed to to access any VoIP service (registration, incoming or outgoing calls).

  * `voip` - if the user belongs to this group, he is allowed to register with the SIP server and make VoIP calls.

  * `pstn` - if the user belongs to this group, he is allowed to register, call to other VoIP users and to PSTN numbers.

```
#
# kamailio with radius config script
#

# ----------- global configuration parameters ------------------------

debug=7            # debug level (cmd line: -dddddddddd)
fork=no
log_stderror=yes    # (cmd line: -E)

check_via=no    # (cmd. line: -v)
dns=no          # (cmd. line: -r)
rev_dns=no      # (cmd. line: -R)
port=5060
children=4
listen=udp:10.10.10.10
alias="kamailio.org"

#fifo="/tmp/openser_fifo"

# ------------------ module loading ----------------------------------
mpath="/usr/local/openser-1.0.1/lib/openser/modules"

loadmodule "mysql.so"
loadmodule "sl.so"
loadmodule "tm.so"
loadmodule "rr.so"
loadmodule "maxfwd.so"
loadmodule "avpops.so"
loadmodule "usrloc.so"
loadmodule "registrar.so"
loadmodule "textops.so"
loadmodule "xlog.so"
loadmodule "uri.so"
loadmodule "acc.so"
loadmodule "auth.so"
loadmodule "auth_radius.so"
loadmodule "group_radius.so"
loadmodule "avp_radius.so"

# ----------------- setting module-specific parameters ---------------

# -- usrloc params --
#modparam("usrloc","db_url","mysql://openser:openserrw@localhost/openser")
modparam("usrloc", "db_mode", 2)

# -- acc params --
modparam("acc", "radius_flag", 1)
modparam("acc", "radius_missed_flag", 2)
modparam("acc", "log_flag", 1)
modparam("acc", "log_missed_flag", 1)
modparam("acc", "service_type", 15)
modparam("acc", "radius_extra", "Sip-Src-IP=$si;Sip-Src-Port=$sp")
modparam("acc|auth_radius|group_radius|avp_radius", "radius_config",
    "/etc/radiusclient-ng/radiusclient.conf")

# -- group_radius params --
modparam("group_radius", "use_domain", 1)

# -- avpops params --
modparam("avpops", "avp_aliases", "day=i:101;time=i:102")

# -- rr params --
# add value to ;lr param to make some broken UAs happy
modparam("rr", "enable_full_lr", 1)

# -------------------------  request routing logic -------------------

# main routing logic

route{

    # initial sanity checks -- messages with
    # max_forwards==0, or excessively long requests
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483","Too Many Hops");
        exit;
    };

    if (msg:len >=  2048 ) {
        sl_send_reply("513", "Message too big");
        exit;
    };

    # check if user is suspended
    if(is_method("REGISTER|INVITE|MESSAGE|OPTIONS|SUBSCRIBE"))
    {
        if (radius_is_user_in("From", "suspended")) {
            sl_send_reply("403", "Forbidden - suspended");
            exit;
        };
    };
    
    # we record-route all messages -- to make sure that
    # subsequent messages will go through our proxy; that's
    # particularly good if upstream and downstream entities
    # use different transport protocol
    if (!method=="REGISTER")
        record_route();

    # subsequent messages withing a dialog should take the
    # path determined by record-routing
    if (loose_route()) {
        # mark routing logic in request
        append_hf("P-hint: rr-enforced\r\n");
        if(is_method("BYE"))
        { # log it all the time
            acc_rad_request("200 ok");
            acc_log_request("200 ok");
        }
        route(1);
    };

    if(is_method("INVITE") && !has_totag())
    {   # set the acc flags
        setflag(1);
        setflag(2);
    };

    if (!uri==myself) {
        # check if user is allowed to do voip calls to other domains
        if(is_method("INVITE|MESSAGE")) {
            if (!radius_is_user_in("From", "voip")) {
                sl_send_reply("403", "Forbidden VoIP");
                exit;
            };
        };
        # mark routing logic in request
        append_hf("P-hint: outbound\r\n"); 
        route(1);
    };

    # if the request is for other domain use UsrLoc
    # (in case, it does not work, use the following command
    # with proper names and addresses in it)
    if (uri==myself) {
        # authenticate registers
        if (method=="REGISTER") {
            if (!radius_www_authorize("kamailio.org")) {
                www_challenge("kamailio.org", "0");
                exit;
            };

            # check the src ip address
            if(!avp_check("i:2", "eq/$src_ip/ig"))
            {
                sl_send_reply("403", "Forbidden IP");
                exit;
            };

            save("location");
            exit;
        };

        # calls to pstn
        if(uri=~"sip:00[1-9][0-9]+@") {
            if(is_method("INVITE") && !has_totag()) {
                if (!radius_is_user_in("From", "pstn")) {
                    sl_send_reply("403", "Forbidden PSTN");
                    exit;
                };
            };
            # set gateway address
            rewritehostport("10.10.10.10:5090");
            route(1);
        };
        
        # load callee's avps
        if(avp_load_radius("callee"))
        {
            # check if user has time filter enabled
            if(avp_check("i:3", "eq/i:1"))
            {
                # print time in an avp
                avp_printf("i:100", "$Tf");
                # extract day
                avp_subst("i:100/i:101", "/(.{3}) .+/*\1*/");
                if(!avp_check("i:6", "fm/$day")) {
                    sl_send_reply("403", "Forbidden - day");
                    exit;
                };
                # extract 'hours:minutes'
                avp_subst("i:100/i:102", "/(.{10}) (.{5}):.+/\2/");
                if((is_avp_set("i:4") && avp_check("i:4", "gt/$time")) 
                || (is_avp_set("i:5") && avp_check("i:5", "lt/$time"))) {
                    sl_send_reply("403", "Forbidden - time");
                    exit;
                };
            };
        };
        
        # native SIP destinations are handled using our USRLOC DB
        if (!lookup("location")) {
            # log to acc as missed call
            acc_rad_request("404 Not Found");
            acc_log_request("404 Not Found");
            sl_send_reply("404", "Not Found");
            exit;
        };
        append_hf("P-hint: usrloc applied\r\n"); 
    };

    route(1);
}

# generic forward
route[1] {
    # send it out now; use stateful forwarding as it works reliably
    # even for UDP2TCP
    if (!t_relay()) {
        sl_reply_error();
    };
    exit;
}
```

## RADIUS Accounting Records ##

For a comparison of your results, take a look at the next example to be sure
your configuration for accounting works properly.

If you have installed FreeRADIUS from Debian package, the accounting data is
stored in `/var/log/freeradius/radacct`, if from sources, then
`var/log/radius/radacct/`.

The SIP accounting is composed of two records: `start` and `stop`. The `start`
event corresponds to the INVITE request (start of the call), and the `stop`
event corresponds to the BYE request (end of the call).

In the `start` event, the `Calling-Station-Id` attribute identifies the origin
of the call and the `Called-Station-Id` attribute identifies the destination
of the call. The duration of the call is given by the difference of the
timestamps in the `stop` and `start` events.

```
Sun Mar 12 17:29:21 2006
	Acct-Status-Type = Start
	Service-Type = Sip-Session
	Sip-Response-Code = 200
	Sip-Method = INVITE
	User-Name = "101@kamailio.org"
	Calling-Station-Id = "sip:101@kamailio.org"
	Called-Station-Id = "sip:102@kamailio.org"
	Sip-Translated-Request-URI = "sip:102@192.168.0.12:5066"
	Acct-Session-Id = "1dbe198c82543fa2@192.168.0.11"
	Sip-To-Tag = "00D0E90101B8_T9513"
	Sip-From-Tag = "111aa0fda452c726"
	Sip-Cseq = "4435"
	Sip-Src-IP = "192.168.0.11"
	Sip-Src-Port = "5068"
	NAS-IP-Address = 127.0.0.1
	NAS-Port = 5060
	Acct-Delay-Time = 0
	Client-IP-Address = 10.10.10.10
	Acct-Unique-Session-Id = "37fb00358437ff4d"
	Timestamp = 1142177361

Sun Mar 12 17:29:28 2006
	Acct-Status-Type = Stop
	Service-Type = Sip-Session
	Sip-Response-Code = 200
	Sip-Method = BYE
	User-Name = "102@kamailio.org"
	Calling-Station-Id = "sip:102@kamailio.org"
	Called-Station-Id = "sip:101@kamailio.org"
	Sip-Translated-Request-URI = "sip:101@192.168.0.11:5068"
	Acct-Session-Id = "1dbe198c82543fa2@192.168.0.11"
	Sip-To-Tag = "111aa0fda452c726"
	Sip-From-Tag = "00D0E90101B8_T9513"
	Sip-Cseq = "3305"
	Sip-Src-IP = "192.168.0.12"
	Sip-Src-Port = "5066"
	NAS-IP-Address = 127.0.0.1
	NAS-Port = 5060
	Acct-Delay-Time = 0
	Client-IP-Address = 10.10.10.10
	Acct-Unique-Session-Id = "597f048f3aa62ca0"
	Timestamp = 1142177368
```

## Troubleshooting ##

Hopefully, most of common issues will be solved by this section. If not, please
send your questions to `sr-users@lists.kamailio.org`.

### error: cannot open shared object file ###

If you get an error like:

```
libradiusclient.so.0: cannot open shared object file: No such file or directory
```

then your radiusclient library is not found by the library loader. You have to
add the directory containing the radiusclient library ("/usr/local/lib" if you
installed from sources) in `/etc/ld.so.conf` and run as root `ldconfig -v`.

### error: no reply from RADIUS server ###

If you get an error like:

```
authentication failed(RC=1): No reply from RADIUS server
```

check if the FreeRADIUS server is running on the host you specified in the
`servers` file of radiusclient-ng library.

### error: athentication failure ###

If you get an error like:

```
authentication failed: Authentication failure
```

the password used by the SIP client was wrong. You have to check if the password
configured to the SIP phone is the same with the one from `users` file of
FreeRADIUS server.

### error: received invalid reply digest from RADIUS server ###

If you get an error like:

```
check_radius_reply: received invalid reply digest from RADIUS server
```

means that the digest value of the message coming from the RADIUS server isn't
correct. This is usually due to different values of shared secrets in client
and server side.

So double-check that corresponding shared secrets you set for radiusclient
library and freeradius server match (case sensitive). Also, pay attention that
FreeRADIUS server has two files where shared secrets for clients can be
defined: `clients` and `clients.conf`. Check both of them and define the secret
only in one.

## Document Validity ##

This documentation is valid for Kamailio (OpenSER) v1.0.1 and FreeRadius v1.1.0.

## References ##

Links to relevant projects and resources for this tutorial.

  * Kamailio (OpenSER) - https://www.kamailio.org

  * FreeRADIUS - http://www.freeradius.org

  * RadiusClient-ng - http://developer.berlios.de/projects/radiusclient-ng/

  * Kamailio User's Mailing List - sr-users@lists.kamailio.org

  * MySQL - http://www.mysql.com