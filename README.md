# macOS Sierra 10.12.6 on HP G62 b26er

<img src="/images/Laptop.jpeg" alt="HP G62 b26er macOS" height="70%" width="70%">

![image](https://raw.githubusercontent.com/cafeed28/hp-g62-b26er-hackintosh/main/images/About.png)

## What works:
 - CPU: natively
 - GPU: Intel HD Graphics 1st-gen: Full QE/CI, brightness control (VGA/HDMI not working because my ATI 5470 dGPU is R.I.P.)
 - LAN: Realtek FE 8100
 - Sleep
 - Audio: Realtek, speakers, headphones, internal mic (external mic untested)
 - Keyboard
 - DVD Drive (at least it opens from OS, but i have scratched lens)
 - Backlight control: DSDT patch, IntelBacklight.kext and patched AppleBacklight.kext, AppleBacklightExpert.kext and DisplayServices.framework

## What does not works:
 - Wi-Fi: Broadcom 4313, need to replace, but i use LAN 'cause it's faster
 - 3D Acceleration by Intel HD Graphics 😿

## What I haven't tested:
 - Webcam
 - Touchpad (it broken)
 - Card reader (i haven't SD cards at home, but according [this](https://www.insanelymac.com/forum/topic/295612-guide-hp-g62-mac-os-x-mavericks) it must work)
 - External mic (same as cardreader)

## Preparing
 - Download these kexts
 - https://github.com/RehabMan/OS-X-Intel-Backlight
 - https://www.insanelymac.com/forum/topic/286092-guide-1st-generation-intel-hd-graphics-qeci
 - https://github.com/RehabMan/OS-X-FakeSMC-kozlek
 - https://github.com/Mieze/RealtekRTL8100
 - https://sourceforge.net/projects/voodoohda
 - https://github.com/acidanthera/VoodooPS2

## Installation:
 - Create installation USB (TODO)
 - Put `AppleIntelHDGraphics.kext, AppleIntelHDGraphicsFB.kext, AppleIntelHDGraphicsGA.plugin, AppleIntelHDGraphicsGLDriver.bundle, AppleIntelHDGraphicsVADriver.bundle` to `YourUSB/Kexts`
 - Put FakeSMC kexts, `IntelBacklight.kext`, `RealtekRTL8100.kext`, `VoodooHDA.kext` `VoodooPS2Controller.kext`, `Lilu.kext`(I'm not sure) to `YourUSB/EFI/CLOVER/kexts/10.12`
 - Install macOS as normal, but when it installed, don't reboot. Utilities -> Terminal ->
`rm -rf /Volumes/YourDisk/System/Library/Extensions/AppleIntelHDGraphics*`
`cp -R /Volumes/YourUSB/Kexts/AppleIntelHDGraphics* /Volumes/YourDisk/System/Library/Extensions`
 - Reboot and it should work
 - Patch DSDT as described next

## DSDT Patching
Before next steps, apply `G62 350m.txt` [patch](https://www.insanelymac.com/forum/topic/293943-guide-hp-g62-dsdt-working-with-speedstep)

### Fixing backlight
 - Extract your ACPI tables (press F4 in Clover menu)
 - Install MaciASL and iasl (i used https://github.com/RehabMan/Intel-iasl)
 - Decompile your DSDT and SSDT:
`iasl -e SSDT*.aml -d DSDT.aml`
`iasl -e DSDT.aml -d SSDT*.aml`
 - Open DSDT.dsl with MaciASL and follow [this](https://github.com/RehabMan/Laptop-DSDT-Patch#rehabmans-laptop-patch-repository)
 - Apply `[igpu] Rename GFX0 to IGPU` patch
 - Go to `Scope (_SB)` (first occurrence) and add below
```
OperationRegion (BRIT, SystemMemory, 0xC0048254, 0x04)
Field (BRIT, AnyAcc, Lock, Preserve)
{
    LEVL, 32
}
OperationRegion (BRI2, SystemMemory, 0xC0048250, 0x04)
Field (BRI2, AnyAcc, Lock, Preserve)
{
   LEV2, 32
}
OperationRegion (BRI3, SystemMemory, 0xC00C8250, 0x04)
Field (BRI3, AnyAcc, Lock, Preserve)
{
    LEVW, 32
}
OperationRegion (BRI4, SystemMemory, 0xC00C8254, 0x04)
Field (BRI4, AnyAcc, Lock, Preserve)
{
    LEVX, 32
}
```
Note: you need to replace all 0xC.... addreses to your BAR0 register.

You can find it as described [here](https://www.insanelymac.com/forum/topic/287133-guide-backlight-brightness-for-intel-80860046-1st-gen-hd-gma-5700mhd) (Step 1)

Or like this if you on linux (you need Memory at ?0000000 (...) **[size=4M]**)

![image](https://raw.githubusercontent.com/cafeed28/hp-g62-b26er-hackintosh/main/images/lspci.png)

 - Go to DD02 and add Name `(_HID, EisaId ("LCD1234"))` like on screenshot

![image](https://raw.githubusercontent.com/cafeed28/hp-g62-b26er-hackintosh/main/images/DD02.png)

 - Go to `Scope (_PR)` and add below
 ```
 Device (PNLF)
{
    Name (_HID, EisaId ("APP0002"))
    Name (_CID, "backlight")
    Name (_UID, 0x0A)

    Method (_BCL, 0, NotSerialized) // _BCL: Brightness Control Levels
    {
        Return (Package (0x12)
        {
            0x12FF, // Level when machine has full power
            0x0396, // Level when machine is on batteries

            0x0132, // Other supported levels
            0x0161,
            0x0191,
            0x01D1,
            0x0218,
            0x0266,
            0x02B0,
            0x0396,
            0x04C8,
            0x065B,
            0x0873,
            0x0B24,
            0x0EBB,
            0x103A,
            0x11FF,
            0x12FF
        })
    }

    Method (_BCM, 1, NotSerialized) // _BCM: Brightness Control Method 
    {            
        Store (0x80000000, LEV2)            
        Store (Arg0, LEVL)
    }

    Method (_BQC, 0, NotSerialized) // _BQC: Brightness Query Current
    {
        Return (^^_SB.PCI0.IGPU.DD02._BQC ())
    }

    Method (_DOS, 1, NotSerialized) // _DOS: Disable Output Switching
    {
        ^^_SB.PCI0.IGPU._DOS (Arg0)
    }
}
```

 - Go to `Method (_WAK, 1, Serialized)` and add
```
Store (0x80000000, LEVW)
Store (0x13121312, LEVX)
```
(i'm not sure)

![image](https://raw.githubusercontent.com/cafeed28/hp-g62-b26er-hackintosh/main/images/WAK.png)

### Fixing sleep
 - Go to `Device (EHC1)` and add `Name(_STA, 0)` like on screenshot

![image](https://raw.githubusercontent.com/cafeed28/hp-g62-b26er-hackintosh/main/images/EHC1.png)

 - Install [this](https://www.tonymacx86.com/threads/hack-backlight-control-with-intelbacklight-acpibacklight-with-brightness-rollback.240929)

## Clover config
 - SMBIOS: `MacBookPro6,2`
 - SSDT:
 - - EnableC2, EnableC4, EnableC6, EnableC7: true
 - - Generate: CStates, PStates: true
 - - DropOEM: false
 - Boot Arguments: -v kext-dev-mode=1 rootless=0 keepsyms=1 debug=0x100
 - KernelAndKextPatches:
 - - KernelLapic: true
 - - KernelPm: true (I'm not sure)

# Credits
 - RehabMan for kexts
 - Mieze for LAN kext
 - acidanthera for VoodooPS2
 - autumnrain, slice2009, zenith432 for VoodooHDA
 - GhostRaider for Arrandale Grpahics guide
 - giofrida for his [guide](https://www.insanelymac.com/forum/topic/295612-guide-hp-g62-mac-os-x-mavericks)
 - mnorthern for his [guide](https://www.insanelymac.com/forum/topic/287133-guide-backlight-brightness-for-intel-80860046-1st-gen-hd-gma-5700mhd)
 - Sainath for his [patch](https://www.insanelymac.com/forum/topic/293943-guide-hp-g62-dsdt-working-with-speedstep)
 - wanderson.passos for patched [AppleBacklight.kext](https://www.tonymacx86.com/threads/hack-backlight-control-with-intelbacklight-acpibacklight-with-brightness-rollback.240929)

# TODO
 - Use WhatereverGreen instead of patched Arrandale kexts, i found guide but i am too lazy now (https://github.com/Goldfish64/ArrandaleGraphicsHackintosh)
 - Use ACPIBacklight instead of IntelBacklight with patched AppleBacklight
 - Upgrade to newer macOS
 - Write kext for Broadcom WiFi ![image](https://cdn.discordapp.com/emojis/941206395246751784.png?size=20)
