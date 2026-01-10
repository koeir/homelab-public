# mkfs Error
When I ran `mkfs.ext4`, this error popped up:
/dev/sda2 is apparently in use by the system; will not make a filesystem here!
I believe that partition *used to be* for my filesystem.

Then I remembered that fdisk wouldn't take effect until after reboot, so I promptly reboot the system so that
the previous partitions would be cleared, and redid the previous steps.
