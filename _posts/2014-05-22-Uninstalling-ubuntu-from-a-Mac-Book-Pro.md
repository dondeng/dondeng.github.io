---
layout: post
title:  "Uninstalling Ubuntu from a Mac Book Pro"
date:   2014-05-22 12:35:35
categories: ubuntu macbook
---

I am not an apple fan-boy, but I admit that I like apple hardware – specifically the Mac Book Pro. For this reason, I usually run 
Ubuntu on my Mac Book Pro. If it was not for iTunes, I probably would not even dual-boot. I can never get over the awesomeness of 
‘sudo apt-get install xxxxx’ in comparison to the apple ‘walled garden’. Recently, I wanted to sell one of the Mac Book Pro’s I had 
so that  I could up-grade to a later model. (Apparently Mac Book Pro’s never grow old!). The person I was selling the laptop to did not 
want Ubuntu in it so I had the odious task of un-installing Ubuntu (12.04). Therein lies the problem. There is a lot of literature on 
the web about how to install Ubuntu on Mac Books since the advent of Mac Books using Intel chips, but there are not a lot of folks 
talking about how to remove Ubuntu. After Googling around, here are the steps that worked for me:

{{ more }}


1. Get an Ubuntu – Live CD. You will need it to delete the Linux partitions
2. Boot into the Ubuntu Live CD (press the C button while restarting with the Ubuntu CD/DVD in the drive)
3. Start Gparted the Ubuntu partitioning program. If unsure on how to do this, just go the the ‘Dash Home’ and search for Gparted
4. Delete the Linux partition
5. Set the swap partition to “off”. In my case, I tried deleting the swap partition and it would not let me do so until I turned it ‘off’
6. Now delete the swap partition
7. Gparted operations are complete at this time
8. Important note! Do NOT re-size the Mac-OS partition while in Gparted this will be done from Mac-OS
9. Reboot the laptop and boot into Mac OS X
10. Start the Disk Utility application and go to ‘Partition’
11. Re-size the Mac OS partition by dragging it to the bottom to take up the space that had been taken up by Ubuntu
12. Re-boot and make sure those changes have taken effect
13. Next we need to delete rEffit (I was using rEffit – but  I understand it is now legacy)
14. Instructions for uninstalling rEffit are located at their site here
    In my case method 1 was the one applicable
    Delete the efi folder
    Delete rEffitBlesser folder in “Library/StartupItems”
15. Reboot and that’s it!!
