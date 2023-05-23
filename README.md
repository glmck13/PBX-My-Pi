# HomePBX
Configure home PSTN gateway with Grandstream ATA and FreePBX
<img src=https://github.com/glmck13/HomePBX/blob/main/drawing.png width=600px>  
## Background
For the past several years I’ve been running Asterisk on a Raspberry Pi 3B equipped with an FXO board from SwitchPi.  The system would listen to incoming calls, and then speak the incoming caller ID aloud with the aid of Polly, the text-to-speech service in AWS.  Problem is, the drivers for the SwitchPi board were compiled for a specific version of Raspbian, which made it impossible to upgrade the OS or Asterisk.  So I figured it was a time for a change! I little Googling landed me on the HT813 ATA from Grandstream, so I set out to build a more robust PSTN gateway for my house by marrying that with the latest version of FreePBX & Asterisk from Sangoma.
## FreePBX build instructions
I started with an old Raspberry Pi 3B I had lying around, and installed the latest version of Raspberry Pi OS using the imager tool. I then assigned a fixed IPv4 address to my Pi (this is done via the dhcpcd menu!) and plugged it into one of the available Ethernet ports on my home router.  I named the system “freepbx.home”.  Next I installed the latest releases of FreePBX and Asterisk following RonR’s script posted on DSLReports.  The process required a sequence of reboots of the Raspberry Pi, but it executed flawlessly!
## Grandstream config
Next step was to configure the Grandstream.  I plugged the WAN port of the ATA into an Ethernet jack on my home router (the ATA behaves as a client over this port),  set a static IPv4 address for the device, and disabled IPv6.  I named the device “grandstream.home” on my router.   I then needed to configure a “trunk”, as FreePBX calls it, between the ATA and PBX. I named the trunk “PSTN”.  Here’s what I set on the Grandstream end:
### BASIC SETTINGS:
+ Unconditional Call Forward to VOIP: User ID: PSTN, Sip Server: freepbx.home, Sip Destination Port: 5060
### FXS PORT
+ Primary SIP Server: freepbx.home
+ SIP User ID: 6200 (this is the Asterisk extension # for the FXS port)
+ Authenticate ID: 6200
+ Authenticate Password: (enter a numeric password; must be same on FreePBX end)
+ SIP Registration: Yes
+ Outgoing call without Registration: Yes
### FXO PORT
+ Primary SIP Server: freepbx.home
+ SIP User ID: PSTN (this is the Asterisk trunk name)
+ Authenticate ID: PSTN
+ Authenticate Password: (enter a numeric password; must be same on FreePBX end)
+ SIP Registration: Yes
+ Outgoing call without Registration: Yes
+ Number of Rings: 2 (set as low as possible, but must allow sufficient time for caller ID to be passed in over PSTN line)
+ PSTN Ring Thru FXS: No (this allows the FXS & FXO ports to operate independently)
+ Stage Method (1/2): 1
## FreePBX Config
Here’s how I created the “PSTN” trunk”:
### Connectivity > Trunks > Add Trunk
General
+ Trunk Name: PSTN
+ Oubound CallerrID: PSTN phone #
+ CID Options: Force trunk CID
+ Asterisk Trunk Dial Options: (left this alone; my system had this set to ‘R’)
   
pjsip Settings, General
+ Username: (left this alone; my system has this set to trunk name, i.e. “PSTN”)
+ Auth username: (ditto)
+ Authentication: Both
+ Secret: (setting on HT813)
+ Registration: Receive (these two settings are necessary so the Grandstream FXO and FXS port registration activities don’t interfere with each other)
+ Context: from-trunk-pjsip-PSTN
### Connectivity > Outbound Routes > Add Outbound Route
Route Settings
Route Name: OutboundPSTN
Route CID: PSTN phone #
Override extension: Yes
Trunk Sequence for Matched Routes: “PSTN”
   
Dial Patterns
+ prepend: blank
+ prefix: 9 (i.e. dial ‘9’ for an outside call)
+ match pattern: XXXXXXXXXX (10 digit calling)

### Connectivity > Inbound Routes > Add Inbound Route
+ Description: InboundPSTN
+ Set destination: Extensions, 6100 (my extension)
## Call announcements
+ /etc/asterisk/extensions_override_freepbx.conf on freepbx.home
+ Raspberry Pi with external speaker
+ ksh, ncat, sox, aws creds, aws-polly.sh
+ ringtones.conf, ringtones.sh
+ crontab: @reboot $HOME/bin/ringtones.sh >/tmp/ringtones.log 2>&1 &
## VoIP client
+ linphone (Linux or Windows)
+ register sip:6100@freepbx.home
## References
I cobbled togther the above config from a variety of different sources on the web.  As it turned out, each of the sources turned out to have some inaccurate or missing info, so it took some trial and error before I got everything working. Here are a few of the links that turned out to be most directionally correct:
1. [Convert Your Analog Phone Line To Digital With Grandstream HT813](https://vitalpbx.com/blog/convert-your-analog-phone-line-to-digital/)
2. [FreePBX and Grandstream HT813](https://community.freepbx.org/t/freepbx-and-grandstream-ht813/87346/8)
3. [Configuring a Grandstream HT503 Device to act as an FXO Gateway](https://wiki.freepbx.org/pages/viewpage.action?pageId=33293313)
