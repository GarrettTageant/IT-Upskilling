Claude-generated roadmap:

## Scope and assumptions

- This roadmap has 8 phases. Each phase ends with projects that have measurable pass criteria.
- The durations assume 6 hours of study per week. The total is 20 weeks (120 hours). Scale the durations linearly with your actual hours.
- Phases 1 and 2 need no Linux skills, so you can start them now. Phase 4 and later need `ip`, `iw`, and systemd skills.

| Phase | Subject | Weeks |
|---|---|---|
| 0 | Lab setup | 1 |
| 1 | RF propagation | 3 |
| 2 | Channel planning and regulations | 2 |
| 3 | IEEE 802.11ax and 802.11be | 3 |
| 4 | WPA3 | 3 |
| 5 | Controller architectures | 3 |
| 6 | Site surveys | 3 |
| 7 | Capstone design | 2 |

## Phase 0: lab setup

You need the following items:

- A Linux host, native or virtual. A virtual host needs USB passthrough for adapters.
- The `mac80211_hwsim` kernel module. It simulates 802.11 radios for protocol and security labs. The module copies each transmitted frame to all enabled radios on the same channel. It does not model path loss, so you cannot use it for RF work. Verify that the module exists with `modinfo mac80211_hwsim`.
- A USB adapter whose in-kernel driver supports monitor mode. For 6 GHz capture, the driver must also support 6 GHz. Verify both properties for the exact chipset before you buy.
- One physical 802.11ax access point (AP). Phase 3 MLO work also needs an 802.11be AP and an 802.11be client. Without that hardware, you can study multi-link operation (MLO) only on paper.
- One controller-managed AP platform for Phase 5. Verify licensing and hardware availability before you commit to a vendor.

Reference documents:

- IEEE 802.11be-2024. IEEE published it on 22 July 2025. The IEEE GET Program provides the PDF at no charge.
- RFC 5415 and RFC 5416 (CAPWAP).
- RFC 8110 (OWE).
- 47 CFR Part 15, Subpart E, §15.407.

## Phase 1: RF propagation (3 weeks)

**Study**

- Wavelength: λ = c / f.
- Units: dB, dBm, dBi, dBd. Convert with dBd = dBi − 2.15.
- EIRP = transmit power − cable loss + antenna gain.
- Free-space path loss (FSPL): FSPL (dB) = 20·log₁₀(d in km) + 20·log₁₀(f in MHz) + 32.44. Doubling the distance adds 6.02 dB.
- Thermal noise: N (dBm) = −174 + 10·log₁₀(bandwidth in Hz). Noise power increases by 3.01 dB for each doubling of channel width.
- SNR, RSSI, and receiver sensitivity per modulation and coding scheme (MCS).
- Reflection, refraction, diffraction, scattering, absorption, multipath, delay spread, Fresnel zones.
- The log-distance path loss model: PL(d) = PL(d₀) + 10·n·log₁₀(d/d₀).
- Antenna types, beamwidth, and polarization.

**Project 1.1: link budget calculator.** Write a Python tool.

- Inputs: frequency, distance, transmit power, antenna gains, cable loss, receiver sensitivity.
- Outputs: FSPL, EIRP, received signal level, fade margin.
- Pass criteria: the tool returns 60.2 dB at 2437 MHz and 10 m, and 66.7 dB at 5180 MHz and 10 m. The tool returns −101.0 dBm as the noise floor for a 20 MHz channel.

**Project 1.2: measured path loss exponent.**

1. Fix one AP on one channel.
2. Record the signal level (`iw dev <interface> link`) at 1, 2, 4, 8, and 16 m. Take at least 3 readings per distance.
3. Fit n by linear regression of signal level against 10·log₁₀(d).
4. Pass criteria: the report states n and R². It explains each deviation from the free-space value n = 2.

**Project 1.3: material attenuation.**

- Measure the loss of at least 3 materials at 2.4 GHz and at 5 GHz.
- Pass criteria: the table lists the mean loss and the standard deviation per material and band. Each cell uses at least 5 readings.

## Phase 2: channel planning and regulations (2 weeks)

**Study**

- 2.4 GHz: channels 1–11 in the United States. Channels 1, 6, and 11 do not overlap at 20 MHz.
- 5 GHz: the U-NII sub-bands, Dynamic Frequency Selection (DFS), the channel availability check, and the non-occupancy period.
- 6 GHz in the United States: 5925–7125 MHz. The band has 59 channels at 20 MHz, 29 at 40 MHz, 14 at 80 MHz, 7 at 160 MHz, and 3 at 320 MHz. The preferred scanning channels (PSC) are 5, 21, 37, 53, 69, 85, 101, 117, 133, 149, 165, 181, 197, 213, and 229.
- 6 GHz power classes: low power indoor (LPI), standard power with Automated Frequency Coordination (AFC), and very low power (VLP).
- Co-channel contention compared with adjacent-channel interference.
- Carrier-sense multiple access with collision avoidance (CSMA/CA) and airtime as the shared resource.
- Transmit power control and minimum data rate settings.

**Project 2.1: United States channel plan reference.**

- Build a table of channel numbers, center frequencies, widths, and DFS status for all three bands.
- Take the limits from §15.407, and record the section number for each limit.
- Pass criteria: the 6 GHz counts equal 59, 29, 14, 7, and 3. The power limits for each 6 GHz class match the regulation text.

**Project 2.2: scan-based channel recommendation.**

- Scan your environment with `iw dev <interface> scan` and with Kismet or `tshark`.
- Compute the AP count per channel, and rank the channels by co-channel contention.
- Pass criteria: the report recommends channels for 2 APs per band. It states that a managed-mode scan observes beacons and probe responses only, and not client airtime.

## Phase 3: IEEE 802.11ax and 802.11be (3 weeks)

**Study**

- 802.11ax (Wi-Fi 6, and Wi-Fi 6E in 6 GHz): OFDMA and resource unit sizes, uplink and downlink MU-MIMO, 1024-QAM, 12.8 µs OFDM symbol with 0.8, 1.6, or 3.2 µs guard interval, BSS color, OBSS PD spatial reuse, target wake time, trigger frames, and the two network allocation vectors.
- 802.11be (Wi-Fi 7): 320 MHz channels in 6 GHz, 4096-QAM, MLO, multi-RU, preamble puncturing, and up to 16 spatial streams.
- PHY rate formula: rate = (N_SD × N_BPSCS × R × N_SS) / T_sym.
  - N_SD is the number of data subcarriers.
  - N_BPSCS is the number of coded bits per subcarrier per stream.
  - R is the coding rate.
  - N_SS is the number of spatial streams.
  - T_sym is the symbol duration including the guard interval.

**Project 3.1: PHY rate calculator.**

- Implement the rate formula for HE and EHT.
- Pass criteria: the tool returns 1201.0 Mbps for HE at 80 MHz, 2 spatial streams, MCS 11, and 0.8 µs guard interval. The tool returns 5764.7 Mbps for EHT at 320 MHz, 2 spatial streams, MCS 13, and 0.8 µs guard interval.

**Project 3.2: capability decode.**

- Capture beacons and association frames from your AP in monitor mode.
- Decode the HE and EHT capability and operation elements.
- Pass criteria: a table lists each field value and its meaning. A computed maximum PHY rate for the actual AP and client pair matches the rate that the client reports.
- If you have 802.11be hardware, also decode the Multi-Link element and identify the MLO modes that each device supports.

**Project 3.3: throughput against PHY rate.**

- Run `iperf3` TCP tests at a fixed location with channel widths of 20, 40, 80, and 160 MHz. Add 320 MHz if you have 802.11be hardware.
- Pass criteria: the report contains at least 5 runs per width. It compares each mean goodput with the computed PHY rate. It explains the difference with MAC overhead and contention.

## Phase 4: WPA3 (3 weeks)

**Study**

- WPA3-Personal: Simultaneous Authentication of Equals (SAE, a Dragonfly key exchange; RFC 7664 describes Dragonfly), hash-to-element (H2E), and SAE-PK.
- Protected management frames (PMF, IEEE 802.11w). WPA3 requires PMF.
- Forward secrecy. SAE does not permit offline dictionary attacks against a passive capture.
- WPA3-Enterprise: 802.1X with EAP. The 192-bit mode uses GCMP-256 and 384-bit elliptic curves.
- Enhanced Open: Opportunistic Wireless Encryption (OWE, RFC 8110).
- 6 GHz rules: only WPA3-Personal, WPA3-Enterprise, and OWE are permitted. WPA2 is not permitted. Transition modes are not permitted.
- Wi-Fi 7 rules according to Cisco and Meraki documentation: Wi-Fi 7 certification requires WPA3 on all bands. MLO requires a common authentication and key management (AKM) suite across links. WPA3-Enterprise requires GCMP-256.
- Known attacks: KRACK (WPA2), Dragonblood (WPA3 implementations), and deauthentication attacks (mitigated by PMF).

**Project 4.1: SAE in the simulated lab.** Create 3 simulated radios and capture all frames on the `hwsim0` interface.

```bash
sudo modprobe mac80211_hwsim radios=3
sudo ip link set hwsim0 up
sudo nmcli device set wlan0 managed no
```

Use this `hostapd` configuration for `wlan0`:

```ini
interface=wlan0
driver=nl80211
ssid=lab-sae
hw_mode=g
channel=6
wpa=2
wpa_key_mgmt=SAE
rsn_pairwise=CCMP
ieee80211w=2
sae_pwe=2
sae_password=<passphrase>
```

Use this `wpa_supplicant` network block for `wlan1`:

```ini
network={
    ssid="lab-sae"
    key_mgmt=SAE
    sae_password="<passphrase>"
    ieee80211w=2
}
```

- Pass criteria: the capture shows Authentication frames with algorithm 3 (SAE) for the commit and confirm exchange. It then shows association and the 4-way EAPOL handshake. The RSN element lists the SAE AKM and has PMF required.
- Confirm the Wireshark field names (`wlan.fixed.auth.alg`, `wlan.rsn.akms.type`) in the display filter reference for your Wireshark version.

**Project 4.2: offline attack comparison on your own equipment.**

- Capture a WPA2-PSK handshake from your own AP, using a weak passphrase. Run a dictionary attack with `aircrack-ng` or `hashcat`.
- Repeat the capture with SAE.
- Pass criteria: the passphrase is recovered from the WPA2 capture. The report explains why the SAE capture does not give the attacker the data for an offline guess.
- Capture and test only networks that you own.

**Project 4.3: WPA3-Enterprise.**

- Build `hostapd` with FreeRADIUS and EAP-TLS. Use `ieee80211w=2`.
- Pass criteria: the RADIUS log shows Access-Accept. The capture shows the AKM suite and the group management cipher. The 192-bit mode capture shows GCMP-256 as the pairwise cipher.

## Phase 5: controller architectures (3 weeks)

**Study**

- Autonomous APs.
- Controller-based split-MAC with CAPWAP (RFC 5415). Control uses UDP 5246 and data uses UDP 5247. DTLS is mandatory for control and optional for data. The 802.11 binding is in RFC 5416.
- Central switching compared with local switching.
- Cloud-managed controller-less systems, where the data plane remains on the AP.
- AP join: controller discovery (broadcast, DHCP, DNS, or static), then join, image check, and configuration.
- Controller high availability.
- Layer 2 and layer 3 roaming: PMK caching, opportunistic key caching, 802.11r, 802.11k, and 802.11v.
- Power over Ethernet classes for 802.11ax and 802.11be APs (IEEE 802.3at and 802.3bt).

**Project 5.1: architecture comparison.**

- Compare autonomous, central-switching, local-switching, and cloud-managed systems.
- Attributes: control plane location, data plane path, management plane, behavior when the controller is unreachable, and scaling limits.
- Pass criteria: every cell cites an RFC or a vendor document.

**Project 5.2: controller-managed WLAN.**

1. Deploy one AP and one controller.
2. Create 2 SSIDs (one WPA3-Personal, one WPA3-Enterprise or OWE) mapped to 2 VLANs.
3. Pass criteria: a client on each SSID receives an address from the correct VLAN. A capture on the AP uplink shows 802.1Q tags in local switching mode. On a CAPWAP platform, a capture in central switching mode shows UDP 5247 encapsulation.

**Project 5.3: controller failure.**

- Disconnect the controller from the AP uplink in each switching mode.
- Record the behavior for associated clients, new associations, and authentication.
- Pass criteria: the results match the vendor documentation. You explain each deviation.

**Project 5.4: roaming time.**

- Use 2 BSSIDs of the same SSID. Force roams with `wpa_cli roam <bssid>` while `ping -i 0.01` runs.
- Test WPA3-SAE without fast transition and with fast transition.
- Pass criteria: the report lists the traffic gap in milliseconds for at least 10 roams per configuration.
- Take timing values from physical hardware. The simulated lab validates frame sequences only.

## Phase 6: site surveys (3 weeks)

**Study**

- Survey types: predictive, AP-on-a-stick pre-deployment, post-deployment validation, passive, and active.
- Requirements gathering: client types, applications, device density, and signal targets. Design targets vary by vendor and application. Cisco's voice design guidance, for example, uses −67 dBm and 25 dB SNR. Verify values in the current vendor guide.
- Capacity planning from airtime.
- Spectrum analysis. It detects non-802.11 interference, which a Wi-Fi adapter cannot detect.

**Project 6.1: requirements document.**

- Pass criteria: the document states client types, per-client throughput, density, and signal targets with sources. It derives the AP count from an airtime calculation.

**Project 6.2: passive survey.**

1. Scale a floor plan.
2. Mark a uniform measurement grid.
3. Record per-BSSID signal levels at each grid point.
4. Build a heat map per band with Python and matplotlib.
5. Pass criteria: the report contains the maps and a coverage analysis against your targets. It also contains an AP placement recommendation and a co-channel overlap analysis.
6. The consumer adapter signal readings are not calibrated. Use them for relative comparison only.

**Project 6.3: prediction validation.**

- Place the AP as recommended and repeat the survey.
- Pass criteria: the report gives the mean absolute error in dB between the predicted and measured levels. It lists the causes of the error.

## Phase 7: capstone design (2 weeks)

Design the WLAN for a three-floor office with 150 users. The users have 802.11ax and 802.11be laptops and 5 GHz voice handsets. The building also has IoT devices that support WPA2 only.

The deliverable contains:

- Link budgets for each band.
- A channel plan for 2.4, 5, and 6 GHz.
- An SSID and security design. It must handle the WPA2-only devices without breaking 6 GHz rules.
- A controller architecture with a failure analysis.
- An AP count with an airtime-based capacity calculation.
- A validation plan.

The pass criterion is a review by an experienced wireless engineer. If you have no reviewer, check each claim against a standard, a regulation, or a vendor document.

## Certification alignment

- CWNP lists CWNA-109 as the current exam. The last day to take it is 31 December 2026. CWNP schedules CWNA-110 for September 2026. I could not confirm that CWNA-110 has been released.
- Independent analysis of the CWNA-109 objectives indicates that the PHY list stops at Wi-Fi 6E and omits 802.11be. Verify this against the objectives on cwnp.com.
- At 6 hours per week, you cannot finish this roadmap before 31 December 2026. Plan for CWNA-110 and read its objectives when you reach Phase 2.
- CWNP recommends, but does not require, 1 year of wireless LAN work experience.

## Limits of this roadmap

- The simulated lab validates protocol and security behavior only. It does not validate RF behavior.
- Without a spectrum analyzer, you cannot identify non-802.11 interference.
- Without 802.11be hardware, Phase 3 MLO remains unverified by practice.
- Controller labs depend on vendor licensing. Verify it before you purchase hardware.
- Lab results do not replace experience on production deployments.
