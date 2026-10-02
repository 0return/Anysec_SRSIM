# AnySec

Functional lab of ANYsec, Nokia's native line-rate encryption on FP5, demonstrated over an Epipe/VLL service transported by SR-ISIS. Everything runs in containers with SR-SIM 25.7.R1 and Containerlab, configured in Classic and MD- CLI

Topology:

<img width="1453" height="334" alt="image" src="https://github.com/user-attachments/assets/e8f8f129-1a1f-458a-bbfa-9512d64b1914" />

What is ANYsec?

ANYsec is Nokia's network encryption solution available on FP5 silicon (since SR OS 23.10.R1). It provides line-rate, low-latency, scalable and quantum-safe encryption that can be applied to any transport (IP, MPLS, Segment Routing, Ethernet, VLAN) and any service, without impacting performance.

How it works under the hood:

It uses IEEE 802.1AE (MACsec) as the datapath encryption engine and IEEE 802.1X (MKA) as the control plane to negotiate and distribute the keys (SAK). MKA is encapsulated over IP/UDP, so peers don't need to be L2-adjacent.
It reuses a MACsec Connectivity Association (CA) with PSK: the MKA key server generates the SAKs locally and distributes them to the peers. Each LSP is encrypted with a unique SAK.
It is anchored to the Segment Routing LSP by node-SID, not to the service or the SDP. That's why any service riding that LSP is encrypted transparently to the client. Supported protocols: SR-ISIS, SR-OSPF and SR-OSPFv3.
On the wire it inserts a MACsec header (ethertype 0x88e5) and an ES (Encryption SID) label between the SR transport label and the service label; the service label (and payload) travel encrypted.
Configuration is bidirectional by design, even for unidirectional LSPs: the outgoing LSP to encrypt and the incoming LSP to decrypt are both identified within the same encryption-group.


Hard requirement: ANYsec needs FP5 + Segment Routing. It does not exist over LDP or RSVP-TE. The only thing you can swap is the IGP that distributes the SIDs (IS-IS or OSPF).


Verification

Run on PE1 (and the equivalent on PE2):



show router isis adjacency                 # adjacency with P1 = Up

show router isis prefix-sids               # 21001 / 21002 / 21100

show router tunnel-table protocol isis     # tunnel to 10.1.1.2 (ANYsec's foundation)

show service sdp                           # Up/Up

show service id 1001 base                  # Epipe Oper Up

show anysec mka-over-ip                    # Operational Status: in-service

show anysec tunnel-encryption detail       # Peer Oper State: Up + counters

show macsec connectivity-association "CA_ANYSEC" detail  # MACsec


On/off test: same ping, encrypted and in clear

Keep one ping running for the whole test and capture on both P1 ports. On SR-SIM a veth capture shows only the ingress direction. Port e1-1-c1-1 gives you PE1 to PE2 traffic, and port e1-1-c2-1 gives you PE2 to PE1 traffic.



Public Wireshark recognizes the frame as MACSEC but does not decode the ANYsec-over-MPLS detail or the payload — which is exactly the proof it's encrypted. For the full breakdown, install the Lua dissectors (anysec-dissectors). SR-SIM note: veth capture only shows the ingress direction; for this point that's enough.


# Step 1: ANYsec ON (baseline)

show anysec tunnel-encryption detail shows Peer Oper State: Up.

In Wireshark the core carries MPLS 21002 / 32101 followed by MACsec (0x88e5) frames.
The icmp and arp filters match nothing. Even the customer's ARP rides inside the encryption.


# Step 2: ANYsec OFF

Shut the peer on both PEs, so that neither side keeps encrypting.


Classic CLI:

# PE1

configure anysec tunnel-encryption encryption-group "EG_ANYSEC" peer 10.1.1.2 shutdown


# PE2

configure anysec tunnel-encryption encryption-group "EG_ANYSEC" peer 10.1.1.1 shutdown

MD-CLI (PE1 shown; on PE2 use peer 10.1.1.1):

edit-config private

/configure anysec tunnel-encryption encryption-group "EG_ANYSEC" peer 10.1.1.2 admin-state disable

commit


Expected results:

show anysec tunnel-encryption detail no longer shows the peer as Up.
The ping keeps working. STP_ANYSEC is configured with no rx-must-be-encrypted, so each PE still accepts unencrypted traffic.
In Wireshark the core shows MPLS 21002 / service label followed by the customer's Ethernet frame. ARP and ICMP between 192.168.100.1 and 192.168.100.2 are fully readable, and the icmp filter matches.
The second label matches the Egress Label in show service id 1001 sdp 12:1001 detail.



<img width="1360" height="474" alt="image" src="https://github.com/user-attachments/assets/f8d0e536-827f-4544-a2b6-1ae0cba6f8d3" />







