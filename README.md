###Process of Modifying DSDT
==========

1.  Download [iasl51](https://bitbucket.org/RehabMan/acpica/downloads) use iasl to extract clean DSDT/SSDTs from raw DSDT/SSDTs
  ```
  ./iasl51 -da *.aml
  // move clean dsl files to clean/ , you can also do it manually!
  mkdir clean && mv raw/*.dsl clean/
  ```
  
2.  Download [MaciASL](https://bitbucket.org/RehabMan/os-x-maciasl-patchmatic/downloads) and start editing clean dsl files. Add [Laptop-DSDT-Patch](https://github.com/RehabMan/Laptop-DSDT-Patch) repo into MaciASL. If you are doing all things correctly, you will get few errors to be fixed in DSDT.dsl (i only got 4 errors).

3. Fix errors (My own errors, I don't know How yours look like): 
  - DSDT: method local variable is not initialized: [Solution Ref #272](http://www.tonymacx86.com/threads/guide-patching-laptop-dsdt-ssdts.152573/page-28#post-1036066)
  - SSDT-0: [Solution Ref #213](http://www.insanelymac.com/forum/topic/290687-wip-hp-envy-17t-j000-quad-haswell-1085109x1010x1011x/?p=1975006)
  - SSDT-3/4: Remove Them! We will generate an SSDT for our own CPU SpeedStepping later.

4. Disable Nvdia:
  - At the top of DSDT, add
  ```
  External (_SB_.PCI0.RP05.PEGP._OFF, MethodObj) // Warning: Unresolved Method, guessing 0 arguments (may be incorrect, see warning above)
  External (_SB_.PCI0.RP05.PEGP._ON, MethodObj) // Warning: Unresolved Method, guessing 0 arguments (may be incorrect, see warning above)
  ```

  - Add `\_SB.PCI0.RP05.PEGP._ON()` at the beginning of `_PTS` method
  - Add `\_SB.PCI0.RP05.PEGP._OFF()` at the end of `_WAK` method, but before `Return` statement
  - Add `\_SB.PCI0.RP05.PEGP._OFF()` at the end of `SB.PCI0._INI` method

5. Graphic Fix:
  - SSDT-0 (Nvidia Graphic fix):
    * Apply Patch to prevent nvidia graphic card overwriting the internal one:
    ```
    into scope label \_SB.PCI0.GFX0 remove_entry;
    into device label WMI1 remove_entry;
    ```
    * Apply Patch `Rename GFX0 to IGPU` -> `Remove _DSM Method`
  - SSDT-1: Apply Patch `Rename GFX0 to IGPU`
  - SSDT-2: Apply Patch `Haswell HD4400` -> `Rename GFX0 to IGPU` -> `Brightness fix Haswell`

6. Shutdown Fix in `DSDT`:
  - Before `_PTS` method, add
  ```
  OperationRegion (PMRS, SystemIO, 0x1830, One)
  Field (PMRS, ByteAcc, NoLock, Preserve)
  {
    , 4,
    SLPE, 1
  }
  ```
  - In `_PTS` method, in `If (LEqual (Arg0, 0x05))` method add:
  ```
  If (LEqual (Arg0, 0x05))
  {
      P8XH (0x04, 0x55, Zero)
      P8XH (0x04, 0x55, One)
      //added to fix shutdown
      Store (Zero, SLPE)
      Sleep (0x10)
      
  }
  ```
  
7. USB Fix in `DSDT` for **El Capitan**: 
  - Apply Patch `USB3 _PRW 0x6D`: Check the description in patch if you're interested. It will fix almost all the USB related stuffs.
  - As Rehabman mentioned, rename EHC* and XHC are not the long term solution. So a patch in Clover 45484331 -> 45483031 is then the tempory solution. It works only with two Kexts provided by Rehabman: `FakePCIID_XHCIMux.kext` + `USBInjectAll.kext`

  
8. Audio Fix
  - Apply Patch `Audio Layout 03`,  works together with AppleHDA/DummyHDA with 03 Layout.
  - Apply Patch `HEPT Fix`, `IRQ Fix`

9. Brightness Keys (*** Only for Synaptics with VoodooPS2 ***) [(Tutorial)](http://www.tonymacx86.com/threads/guide-patching-dsdt-ssdt-for-laptop-backlight-control.152659/)
  - Seems that Keys can't be captured under El Capitain? **Don't** Apple the Patch below ~~Apply patch below: (Notice: _Q8E and _Q8F are what i captured with ACPIDebug.kext, i don't know if yours is same as mine)~~
  ```
  # Make EC-based brightness up/down work with RehabMan VoodooPS2 ACPI keyboard mechanism
  into method label _Q8E replace_content
  begin
  // Brightness Down\n
      Notify(\_SB.PCI0.LPCB.PS2K, 0x0405)\n
  end;
  into method label _Q8F replace_content
  begin
  // Brightness Up\n
      Notify(\_SB.PCI0.LPCB.PS2K, 0x0406)\n
  end;
  
  ```
10. Wait for your contribution!
