# Troubleshooting Windows Wi-Fi Stuck on 2.4 GHz with ASUS Smart Connect

## Problem Summary

I first noticed this problem while playing **Fortnite**.

Fortnite ran normally on my **PlayStation 5**, but on my Windows laptop the connection felt noticeably worse and more prone to lag. Since both devices were using the same home network, this suggested that the problem was probably not my Internet connection itself.

The laptop was using an **Intel Wi-Fi 6E AX211 160 MHz** adapter and connecting through an **ASUS router with AiMesh and Smart Connect enabled**.

The router used a single SSID for both the 2.4 GHz and 5 GHz bands, so devices were expected to automatically connect to the most appropriate band.

The laptop had an excellent Wi-Fi signal, but investigation revealed that it was connecting to **2.4 GHz instead of 5 GHz**.

Before the fix, the laptop's negotiated wireless link rate was approximately:

```text
Receive rate : 287 Mbps
Transmit rate: 287 Mbps
Band         : 2.4 GHz
Channel      : 5
RSSI         : -42 dBm
Signal       : 92%
```

After troubleshooting, the laptop connected to 5 GHz and negotiated:

```text
Receive rate : 1021 Mbps
Transmit rate: 1021 Mbps
Band         : 5 GHz
Channel      : 36
RSSI         : -48 dBm
Signal       : 89%
```

The ultimate cause was the ASUS router automatically selecting **5 GHz channel 64**, which is a DFS channel. After manually changing the 5 GHz radio to **channel 36**, the laptop immediately began seeing and connecting to the 5 GHz network.

---

# 1. Check Which Wi-Fi Band Windows Is Using

Open Command Prompt or PowerShell and run:

```powershell
netsh wlan show interfaces
```

The important fields are:

```text
Band
Channel
Receive rate
Transmit rate
Signal
RSSI
```

My laptop initially reported:

```text
There is 1 interface on the system:

    Name                   : Wi-Fi
    Description            : Intel(R) Wi-Fi 6E AX211 160MHz
    GUID                   : <redacted>
    Physical address       : <redacted>
    Interface type         : Primary
    State                  : connected
    SSID                   : Aegis
    AP BSSID               : <redacted>
    Band                   : 2.4 GHz
    Channel                : 5
    Network type           : Infrastructure
    Radio type             : 802.11ax
    Authentication         : WPA2-Personal
    Cipher                 : CCMP
    Connection mode        : Auto Connect
    Receive rate (Mbps)    : 287
    Transmit rate (Mbps)   : 287
    Signal                 : 92%
    RSSI                   : -42
    Profile                : Aegis
```

This immediately confirmed that the laptop was connected to:

```text
Band    : 2.4 GHz
Channel : 5
```

Even though the signal strength was excellent.

## Understanding RSSI

RSSI is expressed in negative dBm values.

Values closer to zero represent a stronger signal.

A rough reference:

| RSSI | Approximate Quality |
|---:|---|
| -30 dBm | Extremely strong |
| -40 dBm | Excellent |
| -50 dBm | Excellent |
| -60 dBm | Very good |
| -67 dBm | Good |
| -70 dBm | Usable |
| -75 dBm | Weak |
| -80 dBm | Poor |
| -85 dBm | Very poor |

At approximately:

```text
-42 dBm
```

the laptop had more than enough signal strength for a 5 GHz connection.

---

# 2. Configure the Intel Adapter to Prefer 5 GHz

Windows can expose the advanced properties of the Wi-Fi adapter through PowerShell.

Run:

```powershell
Get-NetAdapterAdvancedProperty -Name "Wi-Fi"
```

My Intel AX211 returned properties similar to:

```text
Name  DisplayName                        DisplayValue
----  -----------                        ------------
Wi-Fi Channel Width for 2.4GHz           Auto
Wi-Fi Channel Width for 5GHz             Auto
Wi-Fi Channel Width for 6GHz             Auto
Wi-Fi Transmit Power                     5. Highest
Wi-Fi 802.11n/ac/ax Wireless Mode        4. 802.11ax
Wi-Fi Ultra High Band (6GHz)             Enabled
Wi-Fi MIMO Power Save Mode               Auto SMPS
Wi-Fi Roaming Aggressiveness             3. Medium
Wi-Fi Preferred Band                     3. Prefer 5GHz band
Wi-Fi Throughput Booster                 Disabled
Wi-Fi 802.11a/b/g Wireless Mode          6. Dual Band 802.11a/b/g
```

The important setting was:

```text
Preferred Band : 3. Prefer 5GHz band
```

To show only the most relevant properties:

```powershell
Get-NetAdapterAdvancedProperty -Name "Wi-Fi" |
Where-Object DisplayName -match "Preferred Band|Wireless Mode|802.11"
```

Example output:

```text
Name  DisplayName                     DisplayValue
----  -----------                     ------------
Wi-Fi 802.11n/ac/ax Wireless Mode     4. 802.11ax
Wi-Fi Preferred Band                  3. Prefer 5GHz band
Wi-Fi 802.11a/b/g Wireless Mode       6. Dual Band 802.11a/b/g
```

If the Preferred Band setting is not already configured for 5 GHz, it can usually be changed with:

```powershell
Set-NetAdapterAdvancedProperty -Name "Wi-Fi" `
  -DisplayName "Preferred Band" `
  -DisplayValue "3. Prefer 5GHz band"
```

Depending on the Intel driver version, the exact display value may differ.

You can inspect the available property values in Device Manager if necessary:

```text
Device Manager
→ Network adapters
→ Intel Wi-Fi adapter
→ Properties
→ Advanced
→ Preferred Band
```

---

# 3. Restart the Wi-Fi Adapter

After changing the Preferred Band setting, restart the adapter.

From an elevated PowerShell or Command Prompt:

```powershell
netsh interface set interface "Wi-Fi" disable
netsh interface set interface "Wi-Fi" enable
```

Alternatively, disconnect and reconnect:

```powershell
netsh wlan disconnect
netsh wlan connect name="Aegis"
```

Then check the connection again:

```powershell
netsh wlan show interfaces
```

In my case, the laptop **still connected to 2.4 GHz**.

That indicated that Windows was probably not the actual problem.

---

# 4. Check Which BSSIDs Windows Can See

The next step was determining whether Windows could actually see a 5 GHz version of the network.

Run:

```powershell
netsh wlan show networks mode=bssid
```

Initially, Windows reported only one visible BSSID:

```text
Interface name : Wi-Fi
There are 1 networks currently visible.

SSID 1 : Aegis
    Network type            : Infrastructure
    Authentication          : WPA2-Personal
    Encryption              : CCMP

    BSSID 1                 : <redacted>
         Signal             : 93%
         Radio type         : 802.11ax
         Band               : 2.4 GHz
         Channel            : 5

         Bss Load:
             Connected Stations: 4
             Channel Utilization: 44 (17%)
```

The important part was:

```text
Band    : 2.4 GHz
Channel : 5
```

There was **no 5 GHz BSSID listed at all**.

This changed the diagnosis significantly.

The laptop was not choosing 2.4 GHz instead of 5 GHz.

From Windows' perspective, **there was no 5 GHz version of the network available to choose**.

---

# 5. Check ASUS Smart Connect

The ASUS router was configured with:

```text
Smart Connect: Enabled
Radio Bands:
Tri-Band Smart Connect
(2.4 GHz, 5 GHz-1, and 5 GHz-2)

SSID:
Aegis
```

The Smart Connect steering rules were approximately:

```text
2.4 GHz:
Steer when RSSI is greater than -62 dBm
Target Band: 5 GHz

5 GHz:
Steer when RSSI is less than -82 dBm
Target Band: 2.4 GHz
```

The laptop was receiving approximately:

```text
RSSI: -42 dBm
```

Since:

```text
-42 dBm > -62 dBm
```

the laptop easily satisfied the router's condition for steering a 2.4 GHz client toward 5 GHz.

The Smart Connect thresholds therefore were **not the root cause**.

---

# 6. Check the ASUS 5 GHz Radio Configuration

In the ASUS router interface:

```text
Wireless
→ General
```

the radios were configured approximately as follows:

```text
2.4 GHz
Channel bandwidth: 20/40 MHz
Control Channel: Auto
Current Channel: 5
```

```text
5 GHz-1
Channel bandwidth: 20/40/80 MHz
Control Channel: Auto
Current Channel: 64
```

```text
5 GHz-2
Channel bandwidth: 20/40/80/160 MHz
Control Channel: Auto
Current Channel: 161
```

The important discovery was:

```text
5 GHz-1 Current Control Channel: 64
```

Channel 64 is part of the **DFS spectrum**.

---

# 7. What Are DFS Channels?

DFS stands for:

```text
Dynamic Frequency Selection
```

Certain portions of the 5 GHz Wi-Fi spectrum are shared with radar systems.

Wi-Fi access points using these frequencies must follow additional rules, including detecting radar signals and potentially vacating a channel.

Common DFS channels include frequencies in ranges such as:

```text
52
56
60
64
100
104
108
112
...
```

DFS support is common on modern hardware, including the Intel AX211, but DFS introduces additional complexity.

Depending on the combination of:

- router firmware
- Wi-Fi adapter driver
- regulatory region
- radar detection
- AiMesh behavior
- client compatibility

a client may occasionally fail to discover or reliably use a DFS-based network.

In this case, Windows could not see the ASUS 5 GHz-1 BSSID while it was operating on:

```text
Channel 64
```

---

# 8. Fix: Change the 5 GHz Radio to Channel 36

In the ASUS router:

```text
Wireless
→ General
→ 5 GHz-1
→ Control Channel
```

I changed:

```text
Auto
Current Control Channel: 64
```

to:

```text
36
```

Channels such as the following are non-DFS options commonly suitable for testing:

```text
36
40
44
48
```

I left the channel bandwidth at:

```text
20/40/80 MHz
```

After applying the change and allowing the Wi-Fi radio to restart, the laptop immediately connected to 5 GHz.

---

# 9. Verify the Fix

Run:

```powershell
netsh wlan show interfaces
```

After the change, the laptop reported:

```text
There is 1 interface on the system:

    Name                   : Wi-Fi
    Description            : Intel(R) Wi-Fi 6E AX211 160MHz
    GUID                   : <redacted>
    Physical address       : <redacted>
    Interface type         : Primary
    State                  : connected
    SSID                   : Aegis
    AP BSSID               : <redacted>
    Band                   : 5 GHz
    Channel                : 36
    Network type           : Infrastructure
    Radio type             : 802.11ax
    Authentication         : WPA2-Personal
    Cipher                 : CCMP
    Connection mode        : Auto Connect
    Receive rate (Mbps)    : 1021
    Transmit rate (Mbps)   : 1021
    Signal                 : 89%
    RSSI                   : -48
    Profile                : Aegis
```

The key changes were:

```text
Before:
Band          : 2.4 GHz
Channel       : 5
Receive rate  : 287 Mbps
Transmit rate : 287 Mbps
RSSI          : -42 dBm
```

```text
After:
Band          : 5 GHz
Channel       : 36
Receive rate  : 1021 Mbps
Transmit rate : 1021 Mbps
RSSI          : -48 dBm
```

The negotiated Wi-Fi link rate increased from:

```text
287 Mbps
```

to:

```text
1021 Mbps
```

This is approximately a:

```text
3.6x increase
```

in negotiated PHY rate.

---

# 10. Important: Link Rate Is Not Internet Speed

The following values:

```text
Receive rate
Transmit rate
```

represent the negotiated **Wi-Fi PHY link rate** between the laptop and access point.

They are not equivalent to actual Internet download or upload speeds.

For example:

```text
Link Rate: 1021 Mbps
```

does not mean an Internet speed test will necessarily achieve 1021 Mbps.

Real-world throughput is reduced by:

- Wi-Fi protocol overhead
- retransmissions
- interference
- router processing
- TCP/IP overhead
- Internet connection speed
- AiMesh backhaul
- the remote server
- other network traffic

However, moving from a 287 Mbps 2.4 GHz link to a 1021 Mbps 5 GHz link provides substantially more wireless capacity and can reduce latency and congestion.

---

# Final Diagnosis

The original assumption was that Windows or the Intel Wi-Fi adapter was incorrectly choosing 2.4 GHz.

The investigation showed something different.

Windows was correctly configured to prefer 5 GHz:

```text
Preferred Band: Prefer 5GHz band
```

However:

```powershell
netsh wlan show networks mode=bssid
```

showed that the laptop could only see the 2.4 GHz BSSID.

The ASUS router's 5 GHz-1 radio was operating on:

```text
Channel 64
```

which is a DFS channel.

After manually changing the ASUS 5 GHz-1 radio to:

```text
Channel 36
```

the laptop immediately connected to 5 GHz and negotiated a link rate of approximately:

```text
1021 Mbps
```

instead of:

```text
287 Mbps
```

## Resolution

```text
ASUS Wireless
→ General
→ 5 GHz-1
→ Control Channel
→ 36
```

For this network, leaving the client-facing 5 GHz radio on a stable non-DFS channel prevented the issue from recurring.

---

# Useful Commands

Check current Wi-Fi connection:

```powershell
netsh wlan show interfaces
```

Show all visible networks and BSSIDs:

```powershell
netsh wlan show networks mode=bssid
```

Show Wi-Fi adapter advanced properties:

```powershell
Get-NetAdapterAdvancedProperty -Name "Wi-Fi"
```

Show relevant Intel wireless properties:

```powershell
Get-NetAdapterAdvancedProperty -Name "Wi-Fi" |
Where-Object DisplayName -match "Preferred Band|Wireless Mode|802.11"
```

Prefer 5 GHz:

```powershell
Set-NetAdapterAdvancedProperty -Name "Wi-Fi" `
  -DisplayName "Preferred Band" `
  -DisplayValue "3. Prefer 5GHz band"
```

Restart the Wi-Fi adapter:

```powershell
netsh interface set interface "Wi-Fi" disable
netsh interface set interface "Wi-Fi" enable
```

Disconnect from Wi-Fi:

```powershell
netsh wlan disconnect
```

Reconnect:

```powershell
netsh wlan connect name="Aegis"
```
