---
layout: post
title:  "Defcon 28: Aerospace Village"
date:   2020-08-20 00:00:00 -0400
categories: master
---

I spent a lot of my time during Defcon 28 in the aerospace village, that is, when I wasn't hacking away at the Red Team CTF. This is the first Defcon for aerospace (last year, they were just the aviation village), and this was my first exposure to space and satellites.

I was more interested in the space side of things rather than the aviation side, and below are my rough notes for the talks, workshops, and CTFs that I participated in. 

## Understanding Space Through a Cybersecurity Lense

[Youtube Link](https://www.youtube.com/watch?v=al5eznn4Dh4)

Geostationary/Geosynchronous Orbit: Orbit of satellite matches with revolution of the Earth, so that the satellite appears to 'hover' in the same spot in the sky. ~300 satellites in geosync. orbit
- these satellites can be used to relay messages between ground sations. The ground stations can't communicate with each other due to Earth's curvature, but can communicate to each other if they use a geosync satellite as a middleman.
- these satellites are also used for GPS - Europe has Galileo, China has BeiDou, and Russia has GLONASS
	- with 4 satellites, trilateration is possible
- also used for remote sensing - weather metrics, and monitoring of Earth's environmental state
- need at least 3 sattelites to fully cover the Earth. Even then, there will be no visibility at the poles.

Anatomy of a rocket: "payload" is the machinery needed to achieve the mission; "bus" refers to the supporting infrastructure/machinery ie rocket propulsion

Field of view is smaller than field of regard; determines how many revolutions around the Earth a satellite needs to cover the entire Earth

#### Orbital Mechanics

bigger orbit -> more energy

The Earth drops/curves 5 meters for every km you travel horizontally. We achieve orbit if we can achieve a velocity of 8km per second, or 17,600 mph

Types of orbit: elliptical, parabola, hyperbola. Hyperbola will leave Earth's orbit

Mechanical energy is fixed. Constantly trading kinetic and potential energy but it is conserved. Slow at apogee (far), fast at perigee (close). Angular momentum is also conserved. ISS goes horizon to horizon in 10 minutes.

The larger the period, the more "scrunched" the orbit.
- A satellite with a period of 24 hours will appear stationary
- A satellite with a period of 2.67 hours (smallest period in the diagram) has the biggest orbit

Inclination orbit: angle between the plane of the sattelite's orbit and the equator.
- The ISS's inclination orbit is at ~51.6 degrees so that Russians can reach the ISS. Otherwise, the USA would have preferred an inclination orbit of ~21 degrees (launching from Florida). 

"Low Earth orbit is halfway to anywhere" - once you reach LEO, it takes much less energy to get to anywhere else in space. 8 km/s to get to LEO, 4 km/s to get to anywhere beyond that.

#### Communication Architecture

Spacecraft, ground stations, control center, and relay sattelites. Communication is done via radiofrequency (RF) links.

Deep Space Network (DSN) - NASA's telecommunications system for space missions. International array of giant radio antennas, and 3 facilities on the ground: California, Spain, and Australia, located approx. 120 degrees apart in longitude.

Tracking and Data Relay Satellites (TDRS) - 8 geosyncronous relay satellites for space-based communications. These satellites relay signals between satellites and satellites to ground stations.

#### Threats

Radiation storms, Corona mass ejection (CME), solar flare (electrostatic discharge)

- Interference/damage to the hardware on the satellites because of natural occurrences in space can lead to loss of control/communication to the satellites, leading to a "zombie sattelite" situation. This happened with the Galaxy 15 satellite. It continued to broadcast signals leading to interference to other nearby satellites, and other satellites had to perform evasive maneuvers to avoid colliding with the zombie satellite.

Drag depends on atmospheric density, which depends on the solar "mood", which goes through cycles of high and low activity (solar cycles). More active solar activity -> more atmospheric density -> more energy/fuel is needed. Cycles have a length of about 11 years. We've been enjoying a low-energy cycle for the past decade, but its predicted that we'll enter a high-energy cycle late 2020.

Solar wind - constant "breeze" of charged particles. 

Atomic oxygen - Ultraviolet waves break down oxygen. This oxygen will look to bond with something. This leads to rust on satellites. 

Outgassing (vacuum) - loss of oxygen

"tin whiskers" - crystalline structures that grow on tin finish. Can cause shorting of circuits and system failures. 

heat transfer - biggest source of heat transfer in space is radiation. Electomagmentic radiation (infrared IR), heat is radiated to space.

Space junk/Debris - 20,000+ softball sized-debris or bigger. Evasive maneuvers costs fuel.

EM radiation - sun, photons (light)
- heat on exposed surfaces; ultraviolet can degrade
- radiofreq. interference
- solar pressure - photons don't have mass but have momentum, can change a spacecraft's orientation
	- apparently can sail in space if had a really, really big sail

3 sources of charged particles: solar wind and solar particle events (SPE), galactic cosmic rays, Van Allen radiation belts
- worries with both low energy (plasma) and high energy
	- high energy -> solar particle event, leads to northern & southern lights. Affects semiconductors' ability to conduct, can lead to bit flipping
- spacecraft shielding: most people would assume that lead is a good choice, but its not really. Best is liquid hydrogen and water. Hardware needs to be radiation-tolerant and software needs to do constant error checking.

Vulnerabilities: RF and Datasystems
- the communication subsystem: modulator/demodulator (modem), transmitters, receivers, and antennas; uplink, downlink, forward link, return link, ground links (fiber), cross links (sat-to-sat)
RF modulation: transmits an info signal via a carrier signal. Children's analogy - transmitting voice via 2 cups attached via a string.
	- amplitude modulation or frequency modulation. In space - phase modulation
- space links nowadays should be encrypted. 
- Many cases of jamming, especially in combat zones (i.e. GPS jammers)
- Data Handling System (DHS) - manages ground commands, payload data, telemetry, and patching of software. Has CPU memory, input/output, sensors
- Over time, the cost of the software has become more expensive than cost of hardware
	- Mariner-6 - went to Saturn in 1969 with 30 lines of code.
	- Appolo - 72k of memory
	- more software leads to more threat surface

## Satellite Orbits 101

[Youtube Link](https://www.youtube.com/watch?v=YJsmLlmCG9I)

Location of launch site is often near equator because rotation of earth is fastest at equator. Gives vehicles an extra push/thrust to for orbit
- mountains and valleys affect the gravitational pull on vehicle

Geocentric orbits
- eccentricies: shape of curve
- periodic orbit: closed, repetitive
- escape orbit: "open", uses planet's grav. pull to achieve escape veloc.
- parabolic orbit: minimum energy escape
- hyperbolic orbit: slingshot, vehicle flies by a planet, uses its grav. pull to gain speed
- prograde - same direction as rotation of earth (moving to east)
- retrograde - opposite direction (west), rare because this is more costly

Low Earth Orbit (LEO) - <1240 miles, <= 128 minutes to complete trip around earth, ISS and many other satellites in this orbit, incl. weather satellites

Medium Earth Orbit (MEO) - between 1240 miles and 22,236 mi. 2-24 hours, mostly populated by sats whos purposes are for communication and navigation (GPS, Galileo, GLONASS)
	- geosyn. orbit is 22,236 miles; a 24 hour rotation will make the satellite appear fixed -> geostationary
	- these are often linked to earth antennas (Satellite TV and radio)

High Earth Orbit (HEO) - > 24 hours, satellite is slower than earth and appears like its moving west, but its not retrograde
	- these sattelites are mostly to observe/research space, Hubble

## GPS Spoofing 101

[Youtube Link](https://www.youtube.com/watch?v=mVITimYA3z0)

L1 C/A (legacy) - all sats have this. Modulation - BPSK

31 satellites for GPS @ 12,550 miles (MEO). Orbits are setup such that at least 4 satellites are visible at any point

Transmits nagiv. messages containing location and time of transmission. Each satellite has a unique pseudorandom code for encoding messages. The GPS receiver on ground receives each navig. message and estimates its distance to the satellite to compute the location

Each nagivational message consists of 25 frames. Takes ~12.5 minutes to transmit entire packet. Time info is time from epoch.

![GPS-nagivation-header](https://i.imgur.com/RvFtlPl.png)

Receiver frontend converts analog signals to digital, to be processed by subsequent modules.

Each satellite is assigned a channel
- Acquisition - initial surge, then receiver begins tracking and demodulating the signal
- PVT module - calculates position, velocity, time. Calculates pseudoranges - differences in times of signals between multiple satellites. Vulnerable to spoofing.

#### GPS Spoofing Attacks

- jamming - send noise at higher power than authentic message. Leads to DoS

- spoofing - transmits false info and/or times of arrival, leading to incorrect calculations of location

- Civilian GPS is unencrypted and unauthenticated. Handshake protocols can't be implemented because the receiver has no way to talk back to the satellites.

- "seamless takeover" - adversary starts off by sending lower power signals then increase the power and overshadow the legit signal. Satellite will lock onto the attacker's signal. The adversary can now send modified packets that the satellite will now accept. 

#### Spoofing Detection and Mitigation

- potential solution: PKI and hidden markers (Markus G Kuhn) "An asymmetric security mechanism for nagivation signals"
	- satellites have public keys, recv entire band of signal.
	- transmitter discloses spreading code at a later time, signed by private key.
	- recv verifies signature, then despreads the signal via hidden markers
- another easier, potential solution: use multiple receivers, attacker will have to spoof all the receivers 
- another solution: signal characters to distinguish signal from sattelite in air vs attacker on ground. Fingerprinting

Anti-anti spoofing: replay attacks. INS/GPS sensor fusion

## Talking to Satellites 101

[Youtube Video](https://www.youtube.com/watch?v=JP1Y6QxM1FA)

ISS is 200 miles up, moving at 17,000 mph (10x faster than a bullet). Revolves around Earth in 90 minutes.

Can talk to ISS and astronauts w/ HAM radio and license, though not 24/7 - ISS has a narrow window of communication. If no license, can still listen to transmissions from the ISS, but can't transmit.

AR ISS - Amateur Radio on ISS (education and community driven)

#### What is 2M band, FM, 1200 bps, packet radio?
2 meter packet radio - unattended packet repeater
	- megahertz = million cycles/sec. Wifi is 2.4 GHz
	- speed of light = wavelength * freq -> wavelength = speed of light / freq -> 2.98x10^6/144x10^6 = ~2M wavelength. 2M band is same as saying 144 megahertz.
		- 2M or 144 MHz = VHF (very high freq)

FM - Freqency modulation, signal is carried by changes in frequency

AM - Amplitude modulation, amplitude is modulated up and down; signal is carried by changes in amplitude

1200 bps - bauds not bits, signal interval or pulse
- early modems were 1 bit per baud, but now can do more bits per baud. How? Quadrature Amplitude Modulation (QAM)
	- 9600 bps = 1200 baud per second * 8 bits per baud

What is packet radio? packetized data, instead of continuous flood of data
- APRS: automatic Packet Reporting System, needs transmitter call signal if transmitting. 
	- radio stations repeat signals (digipeat); IE walkie talkie will only send a message so far out ie. nearest radio station, then the radio station will repeat that signal; TTL
		- goal is to reach an IGate, which will send transmission to public internet
- AX.25 (amateur X.25) - call sign included

APRS
- created in 1973, before cellphones
- every region has a different freq.
	- USA = 144.39 MGz
	- Europe = 144.80
	- using same band means lots of potential collisions, so should only send msgs as far as you need to

Can have rasp.pi. do this

Power required to talk to ISS is surprisingly low, 5 watts is enough
- ISS has digirepeater on board

Google manages aprs.fi, there's also ariss.net

Hardware - [github.com/ericEscobar/TalkingToSatellites](https://github.com/ericEscobar/TalkingToSatellites) for an updated list
- less than $100 is possible

Want a directional antennas (more focused) va dish antenna (radiates out)

google "leggios yagi" can create an antenna with pvc pipes and measuring tape (~$10)
- [http://theleggios.net/wb2hol/projects/rdf/tape_bm.htm](http://theleggios.net/wb2hol/projects/rdf/tape_bm.htm)

Tuning the antenna
- SWR - Standing Wave Radio - measures performance
	- good SWR is close to 1:1
- software: rasp.pi., and [direwolf](https://github.com/wb2osz/direwolf), [xastir](https://xastir.org/index.php/Main_Page) - plot where radio transmissions come from

Plan Your Pass
- App for IOS - GoSatWatch, to determine when ISS is overhead
- beware of dopper effect. Frequency will change depending on if ISS is moving toward or away from you -> will need to tune radio

Sample message: KJGOHH>APRS,ARISS,KJGOHH-7:=3660.84N/11879.96W-defcon 28 aerospace village
- KJGOHH - call sign
- APRS,ARISS - want this message to be digipeated
- 7 - indicates using mobile device
- 3660.84N/11879.96W - encoded GPS coords
- "defcon 28..." - the message itself

## Exploiting Spacecraft (using simulation)

[Youtube Link](https://www.youtube.com/watch?v=b8QWNiqTx1c)

Open source solutions: Cosmos, CFE/CFS, OpenSatKit, NOS3

Many targets to go after: ground software, front end processor system (FEP), ICS, operational technology, etc.

C&DH - command and data handling

barriers of entry to space has drastically reduced in the last 5-15 years. Now needs to be secured

User segment (user terminals) - spoofing, DoS, malware; highest likelihood - people & internet access

Ground segment (antennas) - hacking, hijacking, malware

links (RF) - command intrusion, command replay, spoofing

space (satellites) - command intrusion, payload control, DoS, malware

This presentation focuses on replay, command link intrusion (modulate own signal), DoS via GPS Jamming

#### Command Replay Attack

need to sniff signal from ground station to satellite (radiofreq.)

this similution will simulate radiofrequency signals with UDP with tcpdump

the simulation uses real ground software, real flight software, FEP, communication protocols (CCDS, TC/AOS), but no encryption

sim. - "42" by NASA - supports orbital dynamics and environmental models
- capture RF (udp) leaving ground station during command uplink pass and replay that RF (udp) traffic via tcpdump

less than $7k to do RF capture and replay hardware in real life

need signal lock before sending commmand. No OP cmd is satellite equiv. of ping
- command counter gets incremented for each successful command

time to attack is when sat. moves away from ground station (signal lock lost). Ground station won't see anything wrong and the cmd counter increases

#### Command Intrusion

no authentication and encryption means that adversary can craft own raw command packet and uplink it to sat.

CFE/CFS uses CCDS prtocol for uplink/downlink - command space packet
- space packet wrapped in transfer frames, wrapped in Communications Link Transmission Unit (CLTU)
- `(CLTU head)[Transfer Frame Header]<Space Packet>[Transfer Frame Trailer](CLTU Trailer)`

Space Packet Protocol - CCDS 133.0-B-2

[include pic]

Protections
- COP-1 - communications operation procedure-1: adds sequencing protection
		- may be helpful, but what we really need is encryption and authentication

#### DoS Via GPS jamming

block space traffic from GPS sensors via GPS jamming

sattelite goes into sunpoint/safe mode - point solar panels to sun and wait to hear from ground station again
	- lots of functionality is shut down, possibly including protection mechanisms

sometimes uplink is encrypted, but not downlink. Downlink doesn't provide any commands to the satellite, but can still be used to gain telemetry

Protecting the space system (ground to space)
- Very sim. to normal protections, defense in depth. dynamic/static analysis, firewalls/ACLs, segmentation, authentication, IDS/IPs, etc.

## Trust and Truth in Space Situation Awareness

[Youtube Link](https://www.youtube.com/watch?v=rkT8mxbWaIU)

SSA - space situational awareness - tracks debris in space

debris cascade - debris collide with debris, causing more debris, causing more collisions, and so on

SSA as a cyber target
- SSA at core describes orbit of debris, comet, asteroid, sattelites, debris

SSN - Space Surveillance Network
- dominant source of SSA data is US military, who operates the SSN
- comprised of ground stations and some satellites
- most robut, but not cheap, FY15 budget is $1.6 billion

Russian Space Surveillance System
- next best SSA network
- ground stations primarily in Former USSR and territories friendly to USSR
- suspected connections to civil/scientific International Space Observation Network (ISON)

PLA Strategic Support Force (China)
- small compared to US and Russia, limited gobal reach
- expands access via ships

[space-track.org](https://www.space-track.org/)
- publicly avail., offered by USA military

TLE - 2 line element set - primary form of sharing SSA
- started off as a punch card
- serves as input to a propegator -> can predict location of an object in near future
	- accuracy within a kilometer over 72 hours

#### SSA and Trust

few creators of data, everyone else must trust that data -> cyberthreat

why target SSA
- highly centralized databases -> one change can affect many groups/organizations using that data
	- those groups cant verify, must trust implicitly
- soft target, hard effects. AKA easy to hack w/ real effects (ie collisions). Easier&cheaper to hack database versus making own space station

Attack goals
- conceal impending collision
- fake impending collision
	- why fake a collision? adds cost to that organization even if no actual collision occurs - fuel, dollars, mechanical wear and tear, team fatigue)

Mitigations
- vet data where possible. Cross-check public and private data

#### Cyber Security Lessons Learned from Human Spaceflight

[Youtube Link](https://www.youtube.com/watch?v=pieaylw38cY)

ISS - science in microgravity, 24/7 365/yr

Attacks - DoS Jamming, command and communication interception, MITM - alter data stream

~5,000 active satellites in space

#### Architecture

flexibility, redundancy
- had 4 computers loaded w/ Primary Avionics Software System (PASS)
- 1 computer loaded with Backup Flight Software developed completely independently (different company & people, no communication)

distributed architecture separating critical functions
- limit what info is allowed to pass between them, checks to enforce this
- On-orbit Guidance Navigation and Control (GNC) software
- On-orbit Systems Management (SM) software

PGSC - astronauts personal laptops for support
- 1 way communication to GNC, can recv data but not send data to GNC

NISN - NASA integrated Services Network
- entire network owned by NASA, no public internet
	- later on, developed a proxy if astronauts needed/wanted internet access
		- laptops talk to proxy that is firewalled in Johnson Space Center, and the proxy goes to internet
		- station support computers are in no way networked to satellite control systems
- MCC (mission command and control) - Houston texas
- large set of antennas - White Sands
- TDRS - Tracking and Data Relay Satellite (space). Relays to ISS
	- ku-band (video), s-band (audio)

More distributed architecture
- 3 command and control MDMs (multiplexer/demultiplexer)
- multiple separate payload and element MDMs to control systems

Areas of Concern
- encrypted uplinks and downlinks, but onboard satellite bus traffic not encrypted
- ground system vulns
	- used to be each ground system was unique, now theres a push to make them more uniform
		- good: vulnerability in one ground system didn't necessarily mean that other ground systems were also vulnerable
		- bad: patch managament is a nightmare
- edge computing: now there are more command capabilities in the satellite -> more possibility for exploitation (MITM or more severe)
- complacency
- LEO - system of sattelites in a mesh network -> more constant coverage, but more threat surface

Improving Security
- DARPA High-Assurance Cyber Military Systems (HACMS) program
	- hackers successfully hacked the critical systems in a helicopter
	- DARPA System Security Integrated Through Hardware and Firmware (SITTH) program

Delay and Disruption Tolerant Networking - developing a solar system internet

