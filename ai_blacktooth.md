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
	3. PBAP for contacts	

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
	1. Secure aut: only used if both devices have it
	2. Legcay auth

Legacy auth:
	1. Verifier gens AU\_RAND
	2. Claimant receives, gens SRES = H(Kl, AU\_RAND, BD\_ADDR)
	3. Verifier gens SRES; if equal then Kl should be same

Problems:
	1. 
