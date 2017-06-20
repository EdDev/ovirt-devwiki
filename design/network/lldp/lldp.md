# Exposing LLDP information

## Introduction
The Link Layer Discovery Protocol (LLDP) is a vendor-neutral link layer protocol in the Internet Protocol Suite used by network devices for advertising their identity, capabilities, and neighbors on an IEEE 802 local area network, principally wired Ethernet.

This information is usefull for operators and admins to manage and troubleshoot the host networks links, in particular checking the correct connections of the links to the external network entity.

## Requirements

- Expose LLDP TLV information of host interfaces through oVirt REST

- Requirements open questions:
  - At what resolution the LLDP info request is to be supported? One request per cluster/host/iface/tlv?
  - What is the required resolution of the information in relation to time (caching)? If this info is considered 'real time', then no caching is to be applied, the data is valid only when quieried. If the info is to be cached, at what interval it needs to be asked?
  - Are there any requirements in terms of how to format the tlv data in the response?
  - Should we support all possible TLV entires or limit/filter specific ones.

## VDSM implementation options

VDSM is planned to collect LLDP information through LLDPDA or NetworkManager services.
- LLDPDA is as imidiate silution to the data collection as it can be used by VDSM as a driver without any special work dependency.
- In order to use NetworkManager, a preliminary work is required in VDSM to fully integrate it as an active configuration driver, replacing the existing ifcfg usage.

The proposal is to start with LLDPAD as an imidiate solution and continue with NM when it will be integrated into VDSM to replace the ifcfg usage. Then it will become the default driver that collects LLDP information.

### LLDP Report

LLDP information is reported as a property of a NIC in the network caps report.

caps['nics']['nic0']['lldp'] = lldp-info

- lldp-info is a list of TLV entries detected on the nic.
- lldp-tlv entry format is structured as a dict with multiple properties:
  - type-description
  - type: (The ID)
  - subtype: (The ID, may be 0 if it does not exist)
  - oui: Organizationally Unique Identifier
  - value: A list of values for the specified ID. 
  
Note: The oui & subtype are relevant only when the type is 0x7f.

### VDSM - LLDPAD

PoC work has been initiated to report LLDP information using LLDPAD, introducing the LLDPAD driver, its interface and integrating them into the network caps report.

#### Code Module structure

These are the modules involved in the LLDP driver & interface:
vdsm.network.lldpad : LLDPAD driver, using the lldptool as the cli client.
vdsm.network.lldp : LLDP API interface, containing the LLDPAD driver interface (which implements the LLDP API interface using the LLDPAD driver).

#### lldptool report format
~~~~
Chassis ID TLV
	MAC: 30:7c:5e:84:e1:a0
Port ID TLV
	Local: 510
Time to Live TLV
	120
System Name TLV
	rack11-sw02-lab4.tlv
System Description TLV
	Juniper Networks, Inc. ...
System Capabilities TLV
	System capabilities:  Bridge, Router
	Enabled capabilities: Bridge, Router
Port Description TLV
	ge-0/0/2
MAC/PHY Configuration Status TLV
	Auto-negotiation supported and enabled
	PMD auto-negotiation capabilities: 0x0001
	MAU type: Unknown [0x0000]
Link Aggregation TLV
	Aggregation capable
	Currently not aggregated
	Aggregated Port ID: 0
Maximum Frame Size TLV
	9216
Port VLAN ID TLV
	PVID: 150
Unidentified Org Specific TLV
	OUI: 0x009069, Subtype: 1, Info: 504533373135323130333833
VLAN Name TLV
	VID 150: Name vlan-150
LLDP-MED Capabilities TLV
	Device Type:  netcon
	Capabilities: LLDP-MED, Network Policy, Location Identification, Extended Power via MDI-PSE
End of LLDPDU TLV
~~~~
