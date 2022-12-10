## Goal

* Understand process of attack
* Understand vulnerabilities outlined
* Gain deep understanding of how tools used here work
* Replicate attack using Arduino/Raspberry Pi/Android

## Overview

Bl tech [bl special interest group]:
	1. Bl br/edr
	2. Ble

Recent attacks:
	1. Bias -> spoof bl auth [Antonioli]
	2. Knob -> downgrade sess entropy key [Antonioli]
	3. Android+bl profile change trust [Xu]

Blacktooth capabilities:
	1. Impersonate & connect with device w/o confirmation
	2. Bypass auth by downgrade-\>unilateral legacy auth:
	 1. Allows victim device only auth
	3. Force short-key encryption + bruteforce key
	4. Priv esc via profile change

## Background

### Bl BR/EDR

```
Band: 2.4GHz

Piconet:
					M
					|
      -------------------
		|  |  |  |  |  |  |
		s	s	s	s	s	s	s

M[aster] -> clock signal; control s
S[lave]  -> devices
```

Bl core-stack:
	1. Physical
	2. Logical
	3. L2CAP

Bl controller: physical + logical
Bl host: L2CAP + app-protocols in OS
HCI: Host-\>controller comms
SDP: Service discovery protocol; allows sharing of supported services & params
App-layer: implements bl interface & functions for users in userspace

### Bl Pairing

1. M Q>> S for: name & features
	- Any device that starts connection is master
	- Can switch M & S after establishing piconet
2. Secure simple pairing \[SSP] (ECDH) most popular key agreement
	- User confirmation required
	- Link key established afterwards; Link manager protocol [LMP]
	- Can derive encryption key using link key + public params {Entropy: 1-16b}

Problems:
	1. No integrity protection
	2. No encryption
	3. If secure auth[AES CCM] not used then legacy auth for auth, E0 stream enc

### Profile

* Gives functionalities; defined in core stack; SDP advertises these functions

Examples:
	1. A2DP for audio
	2. HID for keystrokes
	3. PBAP for contacts; MAP for messages
	4. AVRCP for audio/video remote control
	5. OPP for file transfer

## Security Analysis of BL BR/EDR

Controller sec:
	1. Auth remote device
	2. Ensure communication confidentiality

Host sec:
	1. Identify class of device
	2. Restrict device capabilities

### Vulnerabilities

* Unfixed roles:
	- Protocol does not define binding rules
	- If attacker pages before the other, then it assumes M role
	- Now bl device (headset) can send commands, strokes (hid), to S

* Unilateral legacy auth:
	- Legacy auth is the only required auth protocol, not sec auth
	- If attacker is M then no auth is required for them, only S
	- Downgrade from sec auth to legacy auth possible by impersonating that the
	  functionality is not supported

* Low encryption key entropy:
	- Real key used to encrypt is derivation of key[link+pparams] via reduction
	  of entropy to N bytes
	- N is result from bl enc key negotiation protocol
	- Attacker impersonates; forces low entropy; bruteforce low entropy key

* Inconsistent profile authentication procedure
	 - bl protocol has terrible profile authentication
	 - No alerts are shown about profile changes
	 - Attacker can force hid even on headsets

* Default profile services authorization
	- All perms granted to arbitrary number of profile requests
	- Headsets allowed to add hid; all perms granted
	- Allows for easy priv esc

## Blacktooth Attack

1. Attacker changes BD\_ADDR to S BD\_ADDR
2. Attacker sends request over bl BR/EDR to pair
3. Attacker forces downgrade to legacy auth
4. Attacker forces low entropy for key, and start bruteforcing for it
5. Attacker connects to sensitive profiles, sends keystrokes via hid profile

### Identity forging and proactive connection request

Forge identity by:
	- Chanigng name of attacker device to S device
	- Grab S BD\_ADDR via eavesdropping or exploit inquiry procedure

Proactive connection request:
	- Abuse ability to be M via first request
	- Headsets do this to auto reconnect; headset is M first, then changes to S
	-  Attacker impersonates & sends request first granting them M role

### Authentication spoofing

Auth methods:
	1. Secure auth: only used if both devices have it
	2. Legacy auth

Legacy auth:
	1. Verifier gens AU\_RAND
	2. Claimant receives, gens SRES = H(Kl, AU\_RAND, BD\_ADDR)
	3. Verifier gens SRES; if equal then Kl should be same

Problems:
	1. Verifier does not have to be M
	2. Unilateral auth, only one needs to auth (S)
	3. Devices paired using sec auth do not need to continue using sec auth
	4. Possible to downgrade & force legacy auth

### Encryption Key Negotiation & Brute Force

Enc methods:
	1. Secure auth key enc
	2. Everything before secure auth

Kc generated from:
	1. AU\_RAND
	2. BD\_ADDR
	3. Kl[shared link key]
	4. Used to:
		- Derive actual key used to encrypt -> Kc~
		- Kc~ entropy level between 1,16 bytes
		- Meaning possible to gen 1 byte entropy

* Key negotiation process implemented in bl controller
	- Victim won't notice anything 

### Profile Change

* Addition of profiles:
	1. M asks S to add new profile
	2. S accepts due to bad bl spec
	3. Mobile OS accepts this change without user confirmation

* Consequence is essentially arbitrary code exec due to ability to upload any
  profile and access all content.

* Attacker cannot read whatever file they want due to that profile requiring
  confirmation from S. But, HID profile allows keystrokes anyways

* Important distinction is that blacktooth does not require pre-installed agents
  on the S device; all this works due to bl spec being terrible

### MITM Blacktooth Variation

* MITM Flow:
	1. Attacker M on two instances; M -> a + M -> b
	2. Share Kl with a & b; not both instances of M
	3. Both instances of M are connected somehow[http?]
	4. M gathers BD\_ADDR & features of a & b
	5. M shares info to opposite device
	6. M initiates auth for both a & b; share auth msg to opposite device
	7. Force small entropy keys for a & b
	8. Bruteforce Kc~
	9. Establish profiles for communicating between both:

* KNOB: Force use of small entropy key
	- [https://github.com/francozappa/knob](Antonioli francozappa/knob)
	- [https://francozappa.github.io/project/knob/](Antonioli vid+exp)

## Implementation & Evaluation

### Attack Implementation

1. Info gathering:
	- Hcidump: get BD\_ADDR, device name, class of device, other att[??] dump
	  to BTSnoop file.
	- Use wireshark to gather info above
	- Change attacker BD\_ADDR to a; send/receive connection to b
	- Niche case 1: some bl device won't respond after being paired but it sends
	  requests to paired device frequently, so if we change to its bd\_addr then
	  we receive its requests and can gather info
	- Niche case 2: sniff bl packets during connection attempt[really expensive]
2. Identity forging & proactive connection request:
	- Use BIAS to create & import impersonation file containing collected data;
	  internalblue used to patch cyw board
	- Scan for victim & send connection request using blctl; M goes to attacker
3. Authentication spoofing:
	- Downgrade to legacy auth via BIAS + internalblue config saying sec auth
	  not supported
	- Configure attack file to:
		- include rom & ram addresses to be patched
		- flag to downgrade sec auth -> legacy auth
		- skip process of verification
4. Encryption key negotiation and brute force:
	- Convice S to set low enc key with low entropy
	- Take l2cap header as oracle[??]
	- Modify cyw board to set lmax and lmin for short key length
	- Attacker computes all possible Kc~ and select according to oracle
	- Reverse engineer where key is stored in RAM attacker side, and place
	  the key there via internalblue
5. Profile change:
	- Implement same profile as device impersonating
	- Use open source[??] to add profiles & complete attackers tasks

## Discussion

### Discoverable & Connectable State

* Victim must be in discoverable & connectable state for attacker to send a con
  request

* Mainstream os's (android, ios, windows, macos) don't have option to turn off
  discoverable but only completely turn off bluetooth

### Profile Change in Linux and Windows

1. Trying to change profile in linux causes connection to be blocked due to
   l2cap\_cr\_sec\_block flag; also sends a security block packet; if a was the
   M instead then profile change would go through
2. Trying to change profile in windows causes PSM not supported packet; a can't
   change profile themselves if M unlike linux

### Defense Against Blacktooth

1. Always ask for user confirmation when connecting to a new bl device;
   functionality implemented in bl host os
2. If sec auth not supported, then legacy auth should force mutual auth instead
   of unilateral auth; meaning both devices have to provide auth not only the S;
   if devices have previously used sec auth then never allow downgrade;
   functionality implemented in bl host os
3. Force entropy of enc keys to be >16B
4. Monitor profile lists, never give full perms to new profiles, notify user
   when new profile is added; functionality implemented in bl host os
