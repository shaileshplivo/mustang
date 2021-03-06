# mustang
# 1. Installation of Freeswitch 1 and 2. 
	root@shaileshpatel-freeswitch1:~#wget -O - https://files.freeswitch.org/repo/deb/debian/freeswitch_archive_g0.pub | apt-key add -
	root@shaileshpatel-freeswitch1:~#echo "deb http://files.freeswitch.org/repo/deb/freeswitch-1.6/ jessie main" > /etc/apt/sources.list.d/freeswitch.list

	# you may want to populate /etc/freeswitch at this point.
	# if /etc/freeswitch does not exist, the standard vanilla configuration is deployed
	root@shaileshpatel-freeswitch1:~#apt-get update && apt-get install -y freeswitch-meta-all
# 2. Installation of Kamailio.
	root@shaileshpatel-kamailio:/usr/local/etc# apt-get install kamailio kamailio-sqlite-modules sqlite3
# 3. Configure Kamailio as registrar.
	create directory for the sqlite database. 
	mkdir /usr/local/etc/kamailio
	
	Configure SQLITE for the database in kamailio. 
	open file /usr/local/etc/kamailio/kamctlrc for editing purpose. 
	Find the DBENGINE variable and set it to SQLITE:
	
	DBENGINE=SQLITE

	And set the DB_PATH variable.
	
	DB_PATH=”/usr/local/etc/kamailio/kamailio.db”

	While editing the kamctlrc file, now would be a good time to also set the variable SIP_DOMAIN. 
	This is not strictly needed to create the database, but will be needed later to actually run Kamailio and have a SoftPhone register with it. 
	The SIP_DOMAIN variable will need to be set to the Kamailio host’s public IP or domain name:

	SIP_DOMAIN=23.253.221.228

	At this point the database can be created with the kamdbctl command. 
	The ownership of the file may also need to be changed to allow the kamailio user read/write access to the database.

	$kamdbctl create
	$chown kamailio:kamailio /usr/local/etc/kamailio/kamailio.db

	Open up the kamailio.cfg file with a prefered editor:

	$ sudo vim /etc/kamailio/kamailio.cfg

	Add the following lines to the top of the kamailio.cfg file:

	#!define WITH_SQLITE
	#!define WITH_AUTH
	#!define WITH_USRLOCDB

	In the Defined Values section add the following block to set the database location for SQLite:

	#!ifdef WITH_SQLITE
	#!ifndef DBURL
	#!define DBURL "sqlite:///usr/local/etc/kamailio/kamailio.db"
	#!endif
	#!endif

	In the Modules Section add the following block to have Kamailio to load the SQLite module:

	#!ifdef WITH_SQLITE
	loadmodule "db_sqlite.so"
	#!endif

	Lastly the alias variable needs to be set to the Kamilio node’s public IP or domain name and port.

	alias=<Kamailio host’s public IP or domain name>:5060

	That about covers the kamailio.cfg file. The next thing to tackle is the kamailio default file:

	$ sudo vim /etc/default/kamailio

	Set the following variables to have the kamailio service run as under the kamailio user and group, and to enable kamailio to run

	RUN_KAMAILIO=yes
	USER=kamailio
	GROUP=kamailio
	
	Add users(subscribers) to the registrar. 
	Username: testplivo1
	Password: plivo0001
	
	Username: testplivo2
	Password: plivo0002
	
	kamctl add testplivo1 plivo0001
	kamctl add testplivo2 plivo0002
# 4. Round-robin scheduling in the kamailio using dispatcher module. 
	http://kamailio.org/docs/modules/4.1.x/modules/dispatcher.html
	edit kamailio config script for the dispatcher module configuration.
	
	create dispatcher.list file which has set 1 and point to two freeswitch server. 
	
	/etc/kamailio/dispatcher.list or /usr/local/etc/kamailio/dispatcher.list.
	
	root@shaileshpatel-kamailio:~# cat /etc/kamailio/dispatcher.list
	# $Id$
	# dispatcher destination sets
	#

	# line format
	# setit(int) destination(sip uri) flags(int,opt) priority(int,opt) attributes(str,opt)

	# freeswitch
	1 sip:23.253.221.87:5060
	1 sip:23.253.221.151:5060

	
	load dispatcher module. 
	
	loadmodule "dispatcher.so"

	configure dispatcher parameters. 
	# ----- dispatcher params -----
	# modparam("dispatcher", "db_url", DBURL)
	# modparam("dispatcher", "table_name", "dispatcher")
	modparam("dispatcher", "flags", 2)
	modparam("dispatcher", "dst_avp", "$avp(AVP_DST)")
	modparam("dispatcher", "grp_avp", "$avp(AVP_GRP)")
	modparam("dispatcher", "cnt_avp", "$avp(AVP_CNT)")
	
	in the route add dispatcher route at the end of route. 
	# dispatch destinations
	route(DISPATCH);
	
	code route for the dispatch requests.
	# Dispatch requests
	route[DISPATCH] {
		# round robin dispatching on gateways group '1'
		if(!ds_select_dst("1", "4"))
		{
			send_reply("404", "No destination");
			exit;
		}
		xlog("L_DBG", "--- SCRIPT: going to <$ru> via <$du>\n");
		t_on_failure("RTF_DISPATCH");
		route(RELAY);
		exit;
	}

	# Sample failure route
	failure_route[RTF_DISPATCH] {
		if (t_is_canceled()) {
			exit;
		}
		# next DST - only for 500 or local timeout
		if (t_check_status("500")
				or (t_branch_timeout() and !t_branch_replied()))
		{
			if(ds_next_dst())
			{
				t_on_failure("RTF_DISPATCH");
				route(RELAY);
				exit;
			}
		}
	}
	
	Restart kamailio. 
	# kamctl restart

	List freeswitch servers. like below for the confirmation of configuration. 
	root@shaileshpatel-kamailio:~# kamcmd dispatcher.list
	{
		NRSETS: 1
		RECORDS: {
			SET: {
				ID: 1
				TARGETS: {
					DEST: {
							URI: sip:23.253.221.151:5060
							FLAGS: AX
							PRIORITY: 0
					}
					DEST: {
							URI: sip:23.253.221.87:5060
							FLAGS: AX
							PRIORITY: 0
					}
				}
			}
		}
	}

# 6. Both freeswitch - Allow sip traffic from the ip address of kamailio. 
	Chagne access control list. and allow traffic from the kamailio to both free switch. 
	
	root@shaileshpatel-freeswitch1:/etc/freeswitch# cat /etc/freeswitch/autoload_configs/acl.conf.xml
	
	<list name="domains" default="deny">
		<!-- domain= is special it scans the domain from the directory to build the ACL -->
		<node type="allow" domain="$${domain}"/>
		<!-- use cidr= if you wish to allow ip ranges to this domains acl. -->
		<!-- <node type="allow" cidr="192.168.0.0/24"/> -->
		<node type="allow" cidr="23.253.221.228/32"/>
	</list>

	vars.xml
	<X-PRE-PROCESS cmd="set" data="internal_auth_calls=false"/>

	add dialplan/public.xml.
	<extension name="from_kamailio">
		<condition field="network_addr" expression="^23\.253\.221\.228" />
		<condition field="destination_number" expression="^(.+)$">
			<action application="transfer" data="$1 XML default"/>
		</condition>
	</extension>

# 7. Playing mp3 files from the http url.
	i did re-installation of the freeswitch from the source. 
	for enabaling mod_shout and mod_v8 module. those are not there in the debian release. 
	
	wget -O - https://files.freeswitch.org/repo/deb/debian/freeswitch_archive_g0.pub | apt-key add -
	echo "deb http://files.freeswitch.org/repo/deb/freeswitch-1.6/ jessie main" > /etc/apt/sources.list.d/freeswitch.list
	apt-get update
	apt-get install -y --force-yes freeswitch-video-deps-most
	 
	# because we're in a branch that will go through many rebases it's
	# better to set this one, or you'll get CONFLICTS when pulling (update)
	git config --global pull.rebase true
	 
	# then let's get the source. Use the -b flag to get a specific branch
	cd /usr/src/
	git clone https://freeswitch.org/stash/scm/fs/freeswitch.git -bv1.6 freeswitch.git
	cd freeswitch.git
	./bootstrap.sh -j
	./configure

	In the module.conf file un-comment mod_shout and mod_v8 for playing mp3 using javascript. then compile & install. 
	
	make
	make install
	
	make cd-moh-install
	make cd-sounds-install

	vim conf/autoload_configs/modules.conf.xml
	enable mp3 module in the modules.conf.xml file. 
	
	<!--For icecast/mp3 streams/files-->
	<load module="mod_shout"/>
	
	<!-- Languages -->
	<load module="mod_v8"/>

	root@shaileshpatel-freeswitch1:/usr/local/freeswitch# cat conf/playmp3.js
	if ( session.ready( ) ) {
	  session.answer( );
	  session.streamFile( "shout://s3.amazonaws.com/plivocloud/Trumpet.mp3" , "");
	  if ( session.ready( ) ) {
		session.hangup( );
	  }
	}

	Add following extension to the default dialplan. in file conf/dialplan/default.xml
	
	<extension name="mp3play">
		<condition field="destination_number" expression="^(12223334444)$">
			<action application="javascript" data="/usr/local/freeswitch/conf/playmp3.js"/>
		</condition>
	</extension>

# 8. bridging between two users. 
	default dialplan extension needs be added as below.
	
	<extension name="bridgetest">
		<condition field="destination_number" expression="^(15556667777)$" break="on-false"/>
			<action application="answer"/>
			<condition field="caller_id_number" expression="^(testplivo1)$" break="on-true">
				<action application="set" data="ringback=${us-ring}"/>
				<action application="set" data="call_timeout=50"/>
				<action application="set" data="continue_on_fail=true"/>
				<action application="set" data="hangup_after_bridge=true"/>
				<action application="export" data="nolocal:absolute_codec_string=PCMA,PCMU"/>
				<action application="bridge" data="{sip_cid_type=none}sofia/internal/testplivo2@10.209.164.165"/>
			</condition>
			<condition field="caller_id_number" expression="^(testplivo2)$" break="on-true">
				<action application="set" data="ringback=${us-ring}"/>
				<action application="set" data="call_timeout=50"/>
				<action application="set" data="continue_on_fail=true"/>
				<action application="set" data="hangup_after_bridge=true"/>
				<action application="export" data="nolocal:absolute_codec_string=PCMA,PCMU"/>
				<action application="bridge" data="{sip_cid_type=none}sofia/internal/testplivo1@10.209.164.165"/>
			</condition>
	</extension>


	Make routing of the numbers in kamailio. following code needs to be changed. 
	# Dispatch requests
	route[DISPATCH] {
		switch($rU){
			case /"^12223334444$":
			case /"^15556667777$":
			case /"^13334445555$":
				# round robin dispatching on gateways group '1'
				if(!ds_select_dst("1", "4"))
				{
						send_reply("404", "No destination");
						exit;
				}
				xlog("L_DBG", "--- SCRIPT: going to <$ru> via <$du>\n");
				t_on_failure("RTF_DISPATCH");
				route(RELAY);
				exit;
			default:
				return;
		}
	}
# 9. Demo Conference should land in previously selected freeswitch, which is based on round robin.
	Both Freeswitch server will run one events python module. this module will listen on the ESL(Event Socket Listener) of each 
	Freeswitch and process events for the demo conference. there are custom events coming for the conference creation and deletion.
	event has Action header which has value conference_create and conference_destroy. 
	in case of conference_create creating new <key,value> pair for the <demo_status,1> and freeswitch ip address from the 
	FreeSWITCH-IPv4 header to <demo,23.253.221.151> in the redis database. redis is installed on the kamailio machine. following is 
	script of the events.py. when conference destroy set <demo_status,0> demo_status to 0. 
	#!/usr/bin/env python
	'''
	events.py - subscribe to all events and print them to stdout
	'''
	import ESL
	import redis

	con = ESL.ESLconnection('localhost', '8021', 'ClueCon')
	r = redis.StrictRedis(host='23.253.221.228', port=6379, db=0)

	if con.connected:
	con.events('plain', 'all')
	while 1:
	e = con.recvEvent()
	if e:
	    #print e.serialize()
	    if e.getHeader("Event-Name") == "CUSTOM":
		if e.getHeader("Event-Subclass") == "conference::maintenance":
			if e.getHeader("Conference-Name") == "demo":
				print e.getHeader("Action")
				if e.getHeader("Action") == "conference-create":
					r.set('demo', e.getHeader("FreeSWITCH-IPv4"))
					r.set('demo_status', 1 )
					print("demo new conference created");

				if e.getHeader("Action") == "conference-destroy":
					r.delete(['demo'])
					r.set('demo_status', 0 )
					print("demo conference destroy");
				'''
				print e.getHeader("Event-Name")
				print e.getHeader("Event-Subclass")
				print e.getHeader("Event-Calling-Function")
				print e.getHeader("Conference-Name")
				print e.getHeader("Action")
				'''

	Use this redis database values of demo_status and demo in the kamailio cfg for the routing of calls for the existing conference. 
	
	from kamailio compile and install ndb_redis module. load ndb_redis module. 

	loadmodule "ndb_redis.so"
	
	initialize the ndb_redis module. connect it for the fetching demo_status and demo values. 
	
	modparam("ndb_redis", "server", "name=redisconf;addr=127.0.0.1;port=6379;db=0")
	
	route the invite whose $rU is pointing to the 13334445555 and whose demo conference is already created on freeswitch 
	server then relay to the same freeswitch server. otherwise follow round robin scheduling as above mention in the 
	route[DISPATCH]. 
	
	route[CONFERENCE] {
		switch($rU){
			case /"^13334445555$":
			if(redis_cmd("redisconf", "GET demo_status", "r")) {
				if($redis(r=>value) > 0){
					if(redis_cmd("redisconf", "GET demo", "r")) {
						$du = "sip:" + $redis(r=>value) + ":" + 5060;
						route(RELAY);
						exit;
					}
				}
			}
			break;
			default:
			break;
			}
		return;
	}

	run events.py using the supervisor. following are steps to install it and use it for the events.py running in background. 
	
	apt-get install supervisor
	
	service supervisor restart

	create a process config file. vim /etc/supervisor/conf.d/events_script.conf. add following content to this fine. 
	
	[program:events_script]
	command=python /usr/src/events.py
	autostart=true
	autorestart=true
	stderr_logfile=/var/log/long.err.log
	stdout_logfile=/var/log/long.out.log

	supervisorctl reread
	
	supervisorctl update
	
	after this, events.py will run in background and supervisor will restart events.py if it crashed because of some unknown
	reason.  

# 10 RTPEngine for the kamailio. 
	i follow instruction from the 
	http://www.sillycodes.com/2015/08/installing-rtpengine-on-ubuntu-1404.html
	to install rtpengine on the debian. current git-master version installed. 
	
	rtpengine --version
	5.0.0.0+0~mr5.0.0.0 git-master-303a40b
	
	kamailio rtpengine configuration. 
	
	#load rtpengine module. 
	loadmodule rtpengine.so
	
	# rtpengine parameter
	modparam("rtpengine", "rtpengine_sock", "udp:23.253.221.228:22222")
	
	# 200 OK or ACK sdp handling. 
	onreply_route[1] {
		if (has_body("application/sdp")) {
			rtpengine_answer("ICE=remove");
		}
	}
	onreply_route[2] {
		if (has_body("application/sdp")) {
			rtpengine_offer("ICE=remove");
		}
	}
	
	# Dispatch requests - intercept invite and ask for new sdp from the rtpengine. for proxy. 
	route[DISPATCH] {
		if (is_method("INVITE|REFER")) {
			if (has_body("application/sdp")) {
				if (rtpengine_offer("ICE=remove")) {
					t_on_reply("1");
				}
			} else {
				t_on_reply("2");
			}
			switch($rU){
				case /"^12223334444$":
				case /"^15556667777$":
				case /"^13334445555$":
					# round robin dispatching on gateways group '1'
					if(!ds_select_dst("1", "4"))
					{
						send_reply("404", "No destination");
						exit;
					}
					xlog("L_INFO", "--- SCRIPT: going to <$ru> via <$du>\n");
					t_on_failure("RTF_DISPATCH");
					route(RELAY);
					exit;
				default:
					return;
			}
		}

		if (is_method("ACK") && has_body("application/sdp")) {
			rtpengine_answer("ICE=remove");
		}
	}

	After above changes in theh kamailio cfg file. kamailio is doing RTP Proxy. and do proxy for the freeswitch media
	as well. 
	
	SRTP Proxy.
	
	TLS configured in the kamailio using following link.
	http://nil.uniza.sk/network-security/tls/configuring-tls-support-kamailio-31-howto
	
	configure listen ip address as tls listener. 
	
	listen=tls:23.253.221.228:5061
	
	define TLS as given below. this will enable the tls.
	
	#!define WITH_TLS
	
	
	http://nil.uniza.sk/network-security/tls/configuring-tls-support-kamailio-31-howto using this. 
	Enable TLS on both freeswitch server. Generate self signed certificate using. 
	root@shaileshpatel-freeswitch1:/usr/local/freeswitch/bin# ./gentls_cert setup -cn 127.0.0.1 -alt DNS:localhost -org 127.0.0.1
	Creating new CA...
	Generating a 2048 bit RSA private key
	..........+++
	................................................................................................................+++
	writing new private key to '/usr/local/freeswitch/conf/ssl/CA/cakey.pem'
	-----
	DONE
	root@shaileshpatel-freeswitch1:/usr/local/freeswitch/bin# ./gentls_cert create_server -cn 127.0.0.1 -alt DNS:localhost -org 127.0.0.1
	Generating new certificate...

	--------------------------------------------------------
	CN: "127.0.0.1"
	ORG_NAME: "127.0.0.1"
	ALT_NAME: "DNS:localhost"

	Certificate filename "agent.pem"

	[Is this OK? (y/N)]
	y
	Generating a 2048 bit RSA private key
	.............+++
	............................................................................+++
	writing new private key to '/tmp/fs-ca-465-20161010045108.key'
	-----
	Signature ok
	subject=/CN=127.0.0.1/O=127.0.0.1
	Getting CA Private Key
	DONE

	configure vars for the tls support. 
	<X-PRE-PROCESS cmd="set" data="internal_tls_port=5061"/>
	<X-PRE-PROCESS cmd="set" data="internal_ssl_enable=true"/>
	<X-PRE-PROCESS cmd="set" data="internal_ssl_dir=$${base_dir}/conf/ssl"/>

	reloadxml. 
	
	and configure the dispatcher.list for the 5061 port. for the tls support. 
	root@shaileshpatel-kamailio:/usr/local/etc/kamailio# cat dispatcher.list
	# $Id$
	# dispatcher destination sets
	#

	# line format
	# setit(int) destination(sip uri) flags(int,opt) priority(int,opt) attributes(str,opt)

	# freeswitch
	1 sips:23.253.221.151:5061
	1 sips:23.253.221.87:5061
	
# 11. Freeswitch running in private network. kamailio as public interface for the sip. 
	Freeswitch runs in the private network and kamailio running on the public network for the sip users. rtpengine is running 
	on the kamailio box for the rtp proxy. following is my dispatch routine in the kamailio.cfg file. 
	# Dispatch requests
	route[DISPATCH] {
		if (is_method("INVITE|REFER")){
			if($Ri == "10.209.164.165"){
				if (has_body("application/sdp")) {
					if (rtpengine_offer("direction=priv direction=pub ICE=remove")) {
						t_on_reply("1");
					}
				} else {
					t_on_reply("2");
				}
				return;
			}
			switch($rU){
				case /"^13334445555$":
					if(redis_cmd("redisconf", "GET demo_status", "r")) {
						if($redis(r=>value) > 0){
							if(redis_cmd("redisconf", "GET demo", "r")) {
								if (has_body("application/sdp")) {
									if (rtpengine_offer("direction=pub direction=priv ICE=remove")){
										t_on_reply("1");
									}
								} else {
									t_on_reply("2");
								}

								$du = "sip:" + $redis(r=>value) + ":" + 5060;
								#$fs = "tls:23.253.221.228:5061";
								$fs = "udp:10.209.164.165:5060";
								route(RELAY);
								exit;
							}
						}
					}
				case /"^12223334444$":
				case /"^15556667777$":
					if (has_body("application/sdp")) {
						if (rtpengine_offer("direction=pub direction=priv ICE=remove")) {
							t_on_reply("1");
						}
					} else {
						t_on_reply("2");
					}

					# round robin dispatching on gateways group '1'
					if(!ds_select_dst("2", "4"))
					{
						send_reply("404", "No destination");
						exit;
					}
					xlog("L_INFO", "--- SCRIPT: going to <$ru> via <$du>\n");
					#$fs = "tls:23.253.221.228:5061";
					$fs = "udp:10.209.164.165:5060";
					t_on_failure("RTF_DISPATCH");
					route(RELAY);
					exit;
				default:
					if (has_body("application/sdp")) {
						if (rtpengine_offer("ICE=remove")) {
							t_on_reply("1");
						}
					} else {
						t_on_reply("2");
					}
			}
		}

		if (is_method("ACK") && has_body("application/sdp")) {
			rtpengine_answer("ICE=remove");
		}
	}

	New freeswitch set added in the dispatcher.list file. these are local interfaces of freeswitches.  
	root@shaileshpatel-kamailio:/usr/local/etc/kamailio# cat dispatcher.list
	# $Id$
	# dispatcher destination sets
	#

	# line format
	# setit(int) destination(sip uri) flags(int,opt) priority(int,opt) attributes(str,opt)

	# freeswitch
	1 sip:23.253.221.151:5060
	1 sip:23.253.221.87:5060

	2 sip:10.209.164.143:5060
	2 sip:10.209.164.82:5060

	in the freeswitch vars.xml file. local_ip_v4 set to ip address of the eth1 interface. 
	<X-PRE-PROCESS cmd="exec-set" data="local_ip_v4=ip addr show eth1 | awk '/inet /{print $2}' | head -n 1 | cut -d '/' -f 1"/>
	
	rtpengine is configured with pub and priv interfaces. and those are used in the kamailio for the sdp processing. 
	
	after these configuration changes public(kamailio) private(freeswitch1 & 2) runs with all above test cases passed. 
	
# Note: I used X-Lite desktop client and Zoiper voip client for the testing purpose. and Wireshark used for the sip/rtp packet analysis. 


	
