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
	1. 
