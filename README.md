## Dell Latitude 7490 OpenCore Configuration

**Best Experienced With: Mac OS Sonoma** or **Mac OS Ventura**

<img width="468" height="285" alt="image" src="https://github.com/user-attachments/assets/f380c944-c267-41e8-9b89-5bfa05c50871" />


First and foremost, this project wouldn’t be here if it wasn’t for this github project:

https://github.com/CloverLeafBG/Dell-Latitude-7490-OC-Hackintosh

This project builds on the previous author’s work, and has been adapted to work with Mac OS Sonoma. AI helped quite a bit, and this contains AI generated code. Wanted to be upfront about it. This has only been tested with one machine, your's might behave differently.

The following boot arguments are all that is needed for me, your experience may vary:
alcid=13 watchdog=0 brcmfx-delay=300
To use DEBUG AirportBrcmFixup.kext add these boot-args:

-brcmfxdbg -liludbgall

**System Configuration**

<img width="468" height="240" alt="image" src="https://github.com/user-attachments/assets/701787bf-ec23-4310-8746-0eb4c663da72" />

**Confirmed Working:**

- USB ports (USBA and USBC)
- HDMI
- Sound
- Brightness and Volume adjust keys function properly
- Sleep works
- Airplay (extensively tested)
- Trackpoint AKA “Pointing Stick”
- Filevault
- Ethernet
- Low Power Mode
- USB PowerShare (Option in bios may be turned on without affecting sleep)
- Graphics Acceleration
- Internal Microphone
- Internal Webcam
- NSS:2 for BCM94360NG

**Not Working 100%**

SD card reader – C State is broken in my case. Must Disable SD card reader so CPU can hit C states C6, C7, and C8. This is a known problem that is being worked on. SD card reader works otherwise.

**Not Tested:**

- Airdrop
- Messages compatibility
- Smart Card Reader
- Fingerprint sensor

## Dell Latitude 7490 BIOS Settings

### General

#### Boot Sequence

- Boot mode: UEFI
- OpenCore: First
- USB Storage Device: Enabled; select through the F12 boot menu when needed

#### Advanced Boot Options

- Enable Legacy Option ROMs: Disabled
- Attempt Legacy Boot: Disabled

#### UEFI Boot Path Security

- UEFI Boot Path Security: Always, Except Internal HDD

### System Configuration

- Integrated NIC: Enabled
- SATA Operation: AHCI
- Drives: Enable all installed drives
- SMART Reporting: Enabled

#### USB Configuration

- Enable USB Boot Support
- Enable External USB Port

#### Thunderbolt Adapter Configuration

- Enable Thunderbolt Technology Support: Enabled
- Security Level: User Authorization

#### Additional Settings

- USB PowerShare: Optional
- Audio: Enable all options
- Unobtrusive Mode: Disabled

### Security

- UEFI Capsule Firmware Updates: Enabled
- TPM 2.0 Security: Enabled
- TPM On: Enabled
- Secure Boot: Disabled
- Secure Boot Mode: Deployed Mode
- Expert Key Management: Disabled
- Intel Software Guard Extensions (SGX): Software Controlled

### Performance

- Multi-Core Support: All cores enabled
- Intel SpeedStep: Enabled
- C-States Control: Enabled
- Intel Turbo Boost: Enabled
- Hyper-Threading Control: Enabled

### Power Management

- USB Wake Support: Enabled
- Block Sleep: Disabled

### POST Behavior

- Fastboot: Minimal

### Virtualization Support

- Virtualization: Enabled
- VT for Direct I/O: Enabled
- Trusted Execution: Disabled


**NSS:2 Fix**
This is probably the most meaningful contribution. The BCM94360NG problem was that on a cold macOS boot, the Broadcom driver applied a single-transmit-chain constraint, leaving the card at NSS:1 instead of NSS:2.

The fix was found by comparing the good NSS:2 state with the bad NSS:1 state (good state initiated by booting into Windows first), then instrumenting and reverse-engineering Apple’s Broadcom driver. That led to the internal function _wlc_stf_txchain_set. Then discovered that it takes four arguments, and logging showed the bad path was specifically setting:  constraint/reason 2 to a chain value of 0x1, whereas the working state used 0x3.

The custom AirportBrcmFixup patch hooks that function and changes only that specific bad case:
reason == 2 && txchain == 1 → change the chain mask to 0x3

Then it lets Apple’s original function continue normally.

So the fix doesn't artificially report NSS:2 or generally force Wi-Fi settings. It prevents Apple’s driver from incorrectly disabling the second transmit chain in that one situation.

The result is that the BCM94360NG comes up with both TX chains enabled and maintains NSS:2 without needing the Windows-boot workaround.

Note: For MacOS Sonoma you must use OpenCore Legacy Patcher in order to get BCM94360NG card functioning, as Sonoma removed driver support for this card. Latest version was used successfully.
Download Here: https://github.com/dortania/Opencore-Legacy-Patcher

**Sleep fix for AlpsHID.kext**
My particular Latitude was failing to fall asleep because the cursor would keep moving slightly, as soon as the lid was closed, thus preventing sleep from occurring. The modified AlpsHID.kext specifically disables cursor input once lid is closed, and resumes functioning when lid is opened again. Using this kext there has been no issue with falling asleep. Previously it was recommended to disable USB PowerShare and USB Wake Support to overcome this specific issue, and it may not be needed anymore if your trackpad can use this kext.
