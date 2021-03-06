# Deprecation Notice
Upon examination and after discussion we have found that the TCN Protocol is identical (in function) to this protocol. In particular, that the STRICT protocol outlined here is a subset of the TCN Protocol, given that the key material to generate new broadcast ids is rotated every time a new broadcast id is required.

We have therefore deprecated it and continue to work on [the common TCN protocol](https://github.com/TCNCoalition/TCN).

# STRICT

STRICT [**s**imply **tr**ace **i**nfe**ct**ions] is a protocol and concept of how to anonymously track infections without tracking people.

The idea for STRICT emerged as part of a grassroots movement during the German government's #WirVsVirus hackathon in March 2020. Its goal is to offer a simple solution for combatting the spread of infective diseases. Our focus is on data minimization, transparency and an easy-to-audit implementation to achieve maximum acceptance.

The current version of this protocol only attempts to solve this problem for the current wave of COVID19 infections. We would like to incorporate the possibility of transferring more information about the type of disease or risk calculation data securly between peers in future versions.

STRICT can be integrated into smart devices with Bluetooth capability, either directly into their operating system or into applications. Through the standardization of this protocol all participants can work together in a cross-border network enabling infection tracking for many geographical locations at once.

## DISCLAIMER

This protocol has not undergone thorough threat modelling and review yet, but we are working on it. It is an open work in progress, since time is of the essence. Any type of constructive feedback is welcome. We are developing a very similar protocol at https://github.com/degregat/ppdt. After the hackathon we copied and altered that protocol draft to match our current design.

## Problem Statement

We want to do privacy preserving contact tracing and notify users if they have come in contact with potentially infected people. This should happen in a way that is as privacy preserving as possible. We want to have the following properties:

- The users should be alerted if they got in touch with infected parties, ideally only that.
- The server should not learn anything besides who is infected, ideally not even that.

## Acronyms

| Short | Long |
| ------------- | ------------- |
| BLE  | Bluetooth Low Energy  |
| TCN  | Temporary Contact Number  |
| N  | # of days of incubation period (+ some margin)  |
| DB  | Database  |

## Protocol Description

- Every participant generates a new random TCN per timeslot (e.g. every 30 minutes), these TCN will be saved with a timestamp and a region tag.
- Each phone is running a BLE (or similar) beacon, broadcasting the current hashed TCN (SHA256 truncated by 6 bytes). If two devices come close to each other, they record each others hashed TCN and save these together with a timestamp, calculated distance (based on signal strength) and meeting duration locally on their device.
- In case of a positive diagnosis for a participant, they submit the randomized history of their TCN from the last N days to a public DB. The TCN of their contacts do not leave their device.
- On the Server, the TCN will be saved with a timestamp for the upload date and a region tag.
- Every participant regularly downloads the new infected TCN from the DB and does a local hash (SHA256) and calculates the intersection with their recorded history and marks them in its own database as infected.
- The users device calculates the risk of the user being infectious in relation to time based on the duration and distance of all TCN marked as infected.
- Recommend actions to the user based on the result of the risk calculation.
- For the times the user was likely to be infectious they publish the respective TCN, and hopefully follow the recommended actions.
- In case of a positive test outcome the user publishes their TCN history and self quarantines. In case of a negative outcome, they continue running the above protocol.

## Possible Extensions

- To exchange bandwidth for post-computation, a ratchet with pre- and post-generation capabilities could be used.
- During contact, if the BLE constraints permit a connection, a key exchange can be performed. Messages encrypted with the resulting key can be appended to the published IDs.
- Saving ICD-10 Codes with uploaded all TCN to calculate the risk of infection separately for every one of them and give specific advice.

## Risk Assessment

- Users log distance and duration for each TCN they see, to calculate risk on device after notification.

## Threat Model

- Clients are assumed to be individually malicious, but not colluding at scale.
- The DB is assumed to be semi-honest.
- We provide only application layer de-correlation. The OS is assumed to be trustworthy, e.g. not recording the Bluetooth MACs; we do not deal with transport for submission and download.

## Malicious Clients

Possible ways for a malicious client to misbehave would be to forge/omit submissions to produce false positives and false negatives. If the TCN are sufficiently long, the collision rate should be low enough to produce few false positives in case of forged submission. Since this is an opt-in protocol, false negatives are identical to non-participation.

A client could correlate TCNs to other users on sidechannels, to later look up which people are positive. This might be mitigated with something like Private Join and Compute, but with malicious security.

## BLE

- BLE4 allows subsequent connections via GATT servers since Android API version 21 (85% of devices according to android studio)
- BLE can exchange 26 bytes without establishing a connection
- BLE ranging seems to be accurate up to 4 meters 
- On Android, Bluetooth MAC rotation on the OS level does not provide further de-correlation, because the MAC address is changed at the same time as the message sent changes.

## Other Layers

- Anonymous submission and anonymous download can further increase user privacy
- Health authorities could give out anonymous credentials for submission with test results if it seems feasible and necessary
- Separating the BT-interaction from the server-interaction, would allow vendors like Apple or Google to solve the Bluetooth interaction and data collectin part of the problem, while still allowing different apps/ vendors to compete on tracing app implementations. This would also allow an app that is running in the background all the time anyway to do the bt data collection, while a different app uses that data to do risk and symptom reporting.

## Privacy and Incentives

- Only the history of TCN of participants who were tested positive is published. Since this history is only correlated to an region, and correlation to contact history happens only on users devices, only regions, but not the contact history is leaked to the server or non-contacts. This, together with voluntary participation, can increase buy-in from the population, leading to faster response time for testing larger groups. Since the health authorities will administer the tests, local statistics will also become more accurate.
- Since people get incentivized to get tested before they become symptomatic, spread can be reduced.
- Since the TCN get rotated, local tracking/correlation by other potentially malicious participants gets impeded.
- This kind of soft, opt-in intervention is probably most useful for the long tail to monitor resurgences. To improve monitoring, the app could walk users through self diagnosis.

## Open Questions

- Does a (weighted) intersection method exist which hides the elements of the intersection from a malicious attacker?
- Which potential malicious user behavior did we miss?
- Can we achieve robustness against colluding clients, for example neighbors or coworkers? Linkage attacks that use location data obtained by stalking Bluetooth emitters are possible.
- Do we need rate limiting to prevent spam on the DB? Can we reduce false positives from forged submissions further this way?
  * Only accept as many TCN as someone could have generated while being infectious. This is probably only possible if an authorization by the health system or a similar party is implemented.
- Do we gain anything from anonymous submission of TCN? (all at once, subsets, individual TCN per circuit or on a mixnet)
  * Solution to the question before would be made ineffective.
- Further analysis of privacy leakage from plaintext DB
- BLE has a range of up to 10 meters. Can we get useful distance information and log it for each TCN of a contact?
  * Yes, if the transmit power and antenna impedance are known. Our sources say its possible to send messages up to 10m indoors, or more if there are no obstructions. Outdoors signals can travel up to 50m and in rare cases even farther [citation needed]. However we have information that the values received via Bluetooth ranging vary a lot based on indoors/outdoors and the amount of water between sender and receiver, e.g. in the human body. Further tests are necessary.
- How long should the TCN be?
- What type and size of regions should be used? For example, would GPS coordinates with 0 or 1 digit precision be sufficient for obscuring users' location?
- [How do we prevent / limit the impact of replay attacs](https://github.com/ito-org/STRICT/issues/3), collecting ids and rebroadcasting them at a different time / location to scare / troll people. Possible aproaches: add (fuzzy?) timestamps to published tokens to prevent rebroadcast at different times. Adding location would prevent rebroadcast at different places, but is contrary to our privacy goals. Switching to key-exchange would neatly prevent this, but is a much more complicated protocol.
