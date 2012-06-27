---
layout: post
title: "Network bonding configuration with Foreman"
category: sysadmin
tags: [foreman, network, HP]
---
{% include JB/setup %}

This post is mostly a note to self. The snippets are kindly provided by user *schewara* who posted them to #theforeman IRC channel. Many thanks for sharing these snippets, they have saved me a lot of time, and hopefully will be usefull for other people as well!

What I was trying to achieve is multi-NIC with bonding configuration in Foreman/Puppet.

One way is to leave this configuration entirelly to Puppet. Another way is to configure the interfaces at the kicstart phase, which is a safer option.

So the snippets are below, verbatim, I will update the post if/when I change them.

Foreman kicstart bit (post install section):

{% highlight bash %}

..
..
SYSTEM_INFO=$(dmidecode -s system-manufacturer)
if [[ "$SYSTEM_INFO" == "HP" ]]; then
  # remove ifcfg Files and udev rules if exist
  rm -f /etc/sysconfig/network-scripts/ifcfg-eth*
  rm -f /etc/udev/rules.d/70-persistent-net.rules
  
  wget -P /etc/yum.repos.d -nc -nd -q http://yum.repo.somewhere/repofiles/3rdParty-hp-spp.repo
  yum install -y -e 0 hpsmh hp-smh-templates hp-health hp-snmp-agents hpdiags hponcfg hpacucli cpqacuxe glibc.i686
  
  modprobe hpilo
  
  # Generate ILO 
  cat > /tmp/ilo-parser.py << EOF
<%= snippet("ILO-IFace-Parser") %>
EOF
  
  <%= @osver <= 5 ? "python26" : "python" -%> /tmp/ilo-parser.py
  
  # CLEAR the IML Log
  hpasmcli -s "clear iml"
  
elif [[ "$SYSTEM_INFO" =~ "VMware" ]]; then
..
..

{% endhighlight %}

Snippet that handles bonding configuration ("ILO-IFacw-Parser"):

{% highlight python %}

#!/usr/bin/env python

# WARNING: Doesnt work with python3

from xml.etree.ElementTree import parse
import sys
import re
import subprocess
import glob

def parse_ilo_interfaces(filename):
#===============================================================================
# Interfaces XML Snipped
# ...
# <SMBIOS_RECORD B64_DATA="0RQA0QADACJkmKx6AAUAImSYrHgAAA==" TYPE="209">
#  <FIELD NAME="Subject" VALUE="Embedded NIC MAC Assignment" />
#  <FIELD NAME="Port" VALUE="1" />
#  <FIELD NAME="MAC" VALUE="00-22-64-98-AC-7A" />
#  <FIELD NAME="Port" VALUE="2" />
#  <FIELD NAME="MAC" VALUE="00-22-64-98-AC-78" />
#  <FIELD NAME="Port" VALUE="iLO" />
#  <FIELD NAME="MAC" VALUE="00-22-64-97-87-8A" />
# </SMBIOS_RECORD>
# ...
#===============================================================================
    ifaces = {}
    port = ""
    
    tree = parse(filename)
    for arecord in tree.findall("SMBIOS_RECORD"):
        if arecord.attrib["TYPE"] == "209":
            childs = arecord.getchildren()
    
    
    for ch in childs:
        if ch.attrib["NAME"] == "Port":
            port = ch.attrib["VALUE"]
        if ch.attrib["NAME"] == "MAC":
            ifaces[port] = ch.attrib["VALUE"]
    
    # remove iLo entry from the dict
    del ifaces["iLO"]
    
    #print(ifaces)
    
    # fix index number and mac formatting
    macp = re.compile('-')
    for id, mac in ifaces.items():
        ifaces[int(id) - 1] = macp.sub(':', mac).upper()
        del ifaces[id]
        
    #print(ifaces)
    #print
    
    return ifaces

def gen_config_params(type, ifacenr="0", ifacemac="", ip="", nm="", gw=""):
    # TODO REFACTOR
    
    if type == "eth":
        cfg_settings = {
                    "DEVICE": type + ifacenr,
                    "HWADDR": ifacemac,
                    "ONBOOT": "yes",
                    "BOOTPROTO": "none",
                    "USERCTL": "no",
                    "NM_CONTROLLED": "no",
                    "SLAVE": "yes",
                    "MASTER": "bond0"
        }
    elif type == "bond":
        cfg_settings = {
                    "DEVICE": type + ifacenr,
                    "ONBOOT": "yes",
                    "BOOTPROTO": "none",
                    "USERCTL": "no",
                    "NM_CONTROLLED": "no",
                    "BONDING_OPTS": "miimon=100 mode=1",
                    "IPADDR": ip,
                    "NETMASK": nm,
                    "GATEWAY": gw
        }
    return cfg_settings

def write_cfgfile(path, ifacelist):
    # Write the file new
    for ifaces in ifacelist:
        print("writing: ifcfg-" + ifaces["DEVICE"])
        
        with open(path + "ifcfg-" + ifaces["DEVICE"], 'w') as f:
            for key, value in ifaces.items():
                f.write(key + '="' + value + '"\n')

def get_ilo_hostinfo(ilooutfile):
    xmlrequest="""<RIBCL version="2.21">
    <LOGIN USER_LOGIN="adminname" PASSWORD="password">
    <SERVER_INFO MODE="READ" >
    <GET_HOST_DATA />
    </SERVER_INFO>
    </LOGIN>
    </RIBCL>"""
    
    subprocess.Popen(["hponcfg", "-i", "-l", ilooutfile], stdin=subprocess.PIPE, stdout=subprocess.PIPE).communicate(xmlrequest)

def parse_udev_interfaces():
    sys_files = glob.glob('/sys/class/net/eth*/address')
    
    maclist = []
    
    for infile in sys_files:
        with open(infile, 'r') as f:
            maclist.append(f.read().rstrip().upper())
    
    for mac in sorted(maclist):
        if mac not in ifacedict.values():
            ifacedict[len(ifacedict)] = mac

if __name__ == '__main__':
    # TODO: call hponcfg and generate XML-File or parse XML Output directly
    path = "/etc/sysconfig/network-scripts/"
    ilooutfile = "/tmp/iloout.xml"
    
    ifacedict = {}
    ifacelist = []
    
    # supplied by Foreman
    ipaddr = '<%= @host.ip -%>'
    netmask = '<%= @host.params["netmask"] -%>'
    gateway = '<%= @host.params["gateway"] -%>'
    
    # testing
    #ipaddr = '192.168.0.10'
    #netmask = '255.255.255.0'
    #gateway = '192.168.0.254'

    get_ilo_hostinfo(ilooutfile)
    ifacedict = parse_ilo_interfaces(ilooutfile)
    
    parse_udev_interfaces()
    
    # Bonding Interface
    cfg_params = gen_config_params("bond", "0", ip=ipaddr, nm=netmask, gw=gateway)
    ifacelist.append(cfg_params)
    
    # Ethernet Interfaces
    for nr, mac in ifacedict.items():
        cfg_params = gen_config_params("eth", str(nr), mac)
        if len(ifacedict) == 4 and nr % 2:
            del cfg_params["SLAVE"]
            del cfg_params["MASTER"]
            cfg_params["ONBOOT"] = "no"
        ifacelist.append(cfg_params)
        
    write_cfgfile(path, ifacelist)

{% endhighlight %}

As a side note, the HP SPP packages are available on [HP site](http://downloads.linux.hp.com/SDR/downloads/SPP/rhel/$release/$arch/current/).
