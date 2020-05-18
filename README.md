# WIP --- Write-up "Dreamcast Japanese Imports" Challenge
This challenge was presented on NSec 2020 and constitute a reverse enginnering challenge of two Dreamcast ROMs.

Challenge author: vincelasal

Write-up author: Joniel Gagné-Laurin (Wizard Hacker) - joniel {[(@)]} gagnelaurin {[(.)]} net

## File list
-   dc_rom_r0.cdi -> Original CDI ROM0 image provided by the challenge
-   dc_rom_r1.cdi -> Original CDI ROM1 image provided by the challenge
-   dc_rom_r0.elf_orig -> Original ELF ROM0 executable provided by the challenge
-   dc_rom_r1.elf_orig -> Original ELF ROM1 executable provided by the challenge

-   dc_rom_r0.elf_mod -> Modified ELF ROM0 to remove the code verification.
-   dc_rom_r1.elf_mod -> Modified ELF ROM1 to remove the code verification.
-   dc_rom_r0n.cdi -> First successful modification of ROM0 with the "Intrusion Detection" message changed to "Wizard Hacker here"
-   dc_rom_r0n2.cdi -> Successful attempt at disabling the code verification of ROM0. Code 000000 provide the flag.
-   dc_rom_r1.cdi -> Successful attempt at disabling the code verification of ROM1. Code 00000 provide the flag.

## Challenge
Riku, the new girl in grade 10 got her hands on some weird Japanese import games for the Dreamcast.
The games want some sort of code, I can’t figure out how to beat them. Could you help me?
If you don’t have a Dreamcast handy, I recommend you play them with [this 4](https://reicast.com/).
-   rom 0
-   rom 0
-   rom 1
-   rom 1
-   [light reading 3](https://www.st.com/content/ccc/resource/technical/document/user_manual/69/23/ed/be/9b/ed/44/da/CD00147165.pdf/files/CD00147165.pdf/jcr:content/translations/en.CD00147165.pdf)
## Introduction
The challenge provide you with 5 files: two .CDI files, two .ELF files and a .PDF. This last file is the instruction documentation for the processor used in the Sega Dreamcast console. The two ELF files are the executables of the game with debug features enabled. The two CDI files are the actual CD images that can be used to play the game.
When booting the game through an emulator, the first ROM ask you to enter a 6 numbers code in order to get the flag. The second ROM is similar, only here it ask you for a 5 numbers code. When you enter a wrong code, the game freeze with the mention Intrusion detected and you need to restart the emulator to try again. This mean that brute-forcing the code is not an option (10-15 seconds per try).
## Disclaimer
I'm not a programmer or reverse engineer. I'm probably going to write something wrong or not understand concept well, nonetheless this is how I did it and understand the challenge.

Also, I'm taking in consideration you have basic knowledge of Linux, Linux compilation, Git, Ghidra, hexadecimal... I'm not providing step procedure for everything, some things you should be able to figure out yourself. If you are stuck, email me and I can help you further.

## Emulator Setup
The challenge provide you a link to the emulator Reicast (https://reicast.com/). You need to Git clone the repository on your machine (did on Ubuntu 18.04), make sure you have all the dependencies specified in the README.md and make the software.

Before you can start a game, you need to retrieve the BIOS of the console. I grab mine from https://replayers.org/bios-files/ and used the "DC - BIOS.bin" and DC - Flash.bin" files. Copy them to "~./.local/share/reicast/data" and rename "DC - BIOS.bin" to "dc_boot.bin" and "DC - Flash.bin" to "dc_flash.bin".

Once you start the emulator (no parameters), you have the option in the application settings to specify the "Content Location". Set it to where you have saved the CDI files. After saving, both CDI should appear. Start either to confirm that your emulator works.

## Code Inspection with Ghidra
Ghidra is a fantastic tool and fortunately know about the architecture of the Dreamcast. You can easily download the software of your computer and import the file. Run through the automatic analysis that it propose you and let it finish the analysis completely before you go further. On the left side, you should have a "Symbol Tree" window, filter for the function "_process_code". Note that I haven't used Ghidra to change the code as it makes other changes to the executables. Instead, I used HexEditor.

On "dc_rom_r0.elf", this should bring you to the function starting at "0x8c010680". To ease visibility, open the decompile code on the left of the listing window. You can see at "0x8c010686" that you have a "comp/eq" operation on "#0x1,r0". Looking at the decompile code, this is a IF instruction that compare a register to "1". If it is true, it jump some instructions and send you toward code that has a reference to "_flag". By modifying the instruction to make sure it always return true, we are making sure to skip the next set of instruction that send you toward "_failure". To do that, let's change the comparison so r0 is compared to r0, always returning true. Looking into the code where we jump, there are also five jumps back to the "_failure" section, so we need to make sure they do not work. This time, instead of changing the operation, let's NOP, or no operation, the jump. So, at lines "0x8c0106a4", "0x8c0106aa", "0x8c0106b2", "0x8c0106ba" and "0x8c0106c0" let's make them a NOP instead.

<img src="https://github.com/jglaurin/NSEC2020-Dreamcast/raw/master/images/ghidra-r0.png">

On "dc_rom_r1.elf", this should bring you to the function starting at "0x8c010720". To ease visibility, open the decompile code on the left of the listing window. You can see at "0x8c010726" that you have a "cmp/eq" operation on "#0x4,r0". Looking at the decompile code, this is a IF instruction that compare a register to "\x04". If it is true, it jump some instructions and send you toward code that has a reference to "_flag". By modifying the instruction to make sure it always return true, we are making sure to skip the next set of instructions that send you toward "_failure". To do that, let's change the comparison so r0 is compared to r0, always returning true. Looking into the code where we jump, there are also three jumps back to the "_failure" section, so we need to make sure they do not work. This time, instead of changing the "cmp/eq" instruction before, let's just NOP, or no operation, the jump. So, at lines "0x8c010748", "0x8c10750" and "0x8c10760" let's make them a NOP instead.

<img src="https://github.com/jglaurin/NSEC2020-Dreamcast/raw/master/images/ghidra-r1.png">

## Modification of the executables
My favorite hex editor is "hexeditor" or the package ncurses-hexedit on Ubuntu, use the one you are the most comfortable with. Having Ghidra open on one screen and hexeditor on the other, I went through both files to change the executables. Note that you will not find the instructions at the same location. The ELF has a header stating that it has to start at "0x8c010000", not "0x0". On hexeditor, it will start at "0x0" so you need to adjust. Also, there is an additional part on top of the code, meaning that "0x8c010000" is actually "0x80" in hexeditor. Before making any changes, situate yourself carefully and make sure you know this is the right location. You can facilitate this by using the bytes windows in Ghidra and comparing the location. Also, I used Ghidra to make the changes temporarily to see what bits were changes, but Hexeditor was used to actually change it on the files.

For "dc_rom_r0.elf", we start at location "0x8c010686" in Ghidra or "0x706" in hexeditor. In the original file, the instruction is "01 88", but we need to change it to "00 30" so it compare r0 to r0. Once completed, let's move to "0x8c0106a4" or "0x724" to replace the jump instruction for a NOP. This means changing "f1 89" to "09 00". Repeat the same for "0x8c0106aa", "0x8c0106b2", "0x8c0106ba" and "0x8c0106c0" so the instructions are all NOP.

For "dc_rom_r1.elf", we start at location "0x8c010726" in Ghidra or "0x7a6" in hexeditor. In the original file, the instruction is "04 88", but we need to change it to "00 30" so it compare r0 to r0. Once completed, let's move to "0x8c010748" or "0x7c8" to replace the jump instruction for a NOP. This means changing "ef 8b" to "09 00" Repeat the same for "0x8c10750" and "0x8c10760 so the instructions are all NOP.

## Dreamcast Development Environment Setup
Now that the executables has been changed, we need to setup the environment to adapt the ELF to play on the Dreamcast. Some steps might be superfluous, but it worked when I executed all of them. For this, let's use the KallistiOS environment. Follow this guide https://dcemulation.org/index.php?title=Compiling_KOS_on_Linux, but you don't need to do the "KOS-Ports" section. On my setup, I didn't used the automatic script, I went through the manual process.

## Modify the executables for packaging
In order for the games to be executables on the console, many sites say that you need to go through an objcopy and scramble process. In addition, I also added a strip process to begin with as the executable I generated the first time was way bigger than the one in the CDI. This is explained by that the ELF contain debug information (I believe the symbols), so it is bigger. The strip process remove that from the executable and I ended up with an executable with a size very close to the one in the CDI.

The first process is to strip the debug information, use the "/opt/toolchains/dc/sh-elf/bin/sh-elf-strip" executable with the option "-s" to remove all debug and relocation information. Then, use the "/opt/toolchains/dc/sh-elf/bin/sh-elf-objdump" executable to transform in a binary the ELF. Make sure you use the "-O binary" option when executing. Finally, you need to scramble the executable with "/opt/toolchains/dc/kos/utils/scramble/scramble" executable. I believe it isn't build through the main build process, so move into the folder and "make" the executable before using it. After scrambling is completed, it should provide you with a binary file that can be packaged into a CDI and executed in the emulator. Repeat this process for each ELF.

## Package the executables
For this part of the process, I used Windows as there is a set of tools to "simplify" the process. Transfer the CDIs and scrambled binaries onto Windows to continue. Then download this package (https://dreamcast-talk.com/forum/download/file.php?id=2682) containing multiple tools that will be used to extract the files in the CDI. Extract the files off the zip and follow the instructions, one ROM at a time. Also, don't mix the ROMs and CDI, although I wouldn't be surprised they can be interchanged.

Copy the CDI file into the "cdi_toolkit/cdirip" folder. Open a CMD line into the folder and execute "cdirip.exe". When prompted, select the CDI file and then select the destination folder, use "cdi_toolkit/isofix". After "cdirip.exe" has completed, in the output you will find a LBA entry for Session 2 (here it is 11702), take note. Move to the "cdi_toolkit/isofix" folder and confirm that you have a file called "TData02.iso" present. Execute "isofix.exe" and enter "TData02.iso" when asked in the CMD prompt. It will also ask you the LBA value you took note earlier, enter it. You will then find that you have new files in the folder, the important one is fixed.iso. In the folder "cdi_toolkit/isobuster", install IsoBuster on the machine to continue the process. Once installed, start IsoBuster and open "cdi_toolkit/isofix/fixed.iso". In the left panel, select the red icon under Session 1, Track 01 and highlight the two files on the right panel. Right-click and select Extract Objects and select the destination as "cdi_toolkit/discroot". At the end of this, you should have two files in "cdi_toolkit/discroot": ip.bin and 1ST_READ.BIN. Delete the 1ST_READ.BIN file and replace it by the altered binary we worked before. Rename the file to 1ST_READ.BIN. 

Download BootDreams here (https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/bootdreams/bootdreams_106c.exe) and execute the installer. This software will recreate the CDI file with the new executable we have modified. Once installed, launch the application and make sure it is set to create a DiskJuggler file (first icon to the left). In "Selfboot folder", browse to the "cdi_toolkit/diskroot" folder, change the cd label if you wish to, make sure the disk format is at Audio\Data and click process. It will ask you where you want to save the new CDI file and bundle the new executable.

Repeat this whole process for the second ROM. At the end, you should have two CDI files, transfer them to your Linux machine. It is just a matter of starting each ROM and press enter at the code entry to get the flag.

## Links
https://dreamcast-talk.com/forum/viewtopic.php?f=52&t=8045&sid=158d552fb143100e581ae4fd70bd1ac7

https://dreamcast.wiki/Creating_a_bootable_Dreamcast_disc

https://github.com/reicast/reicast-emulator

https://replayers.org/bios-files/

https://dreamcast.wiki/Building_KOS_on_Linux_mint_(or_Ubuntu)

https://dcemulation.org/index.php?title=Compiling_KOS_on_Linux

http://gamedev.allusion.net/softprj/kos/

https://github.com/arthabus/DreamcastCdiTool

https://dcemulation.org/index.php?title=BootDream
