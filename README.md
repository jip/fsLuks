# fsLuks
{build,open,print,close} LUKS partition in {device,file}

Filesystem types supported: iso9660, ext4, vfat, swap.

## How to configure (a sketch)

Say, we are going to encrypt a home and a swap. Let the disk layout be as follows:
```text
device name    device UUID                             mount point    description
/dev/sda1      11111111-1111-1111-1111-111111111111    /              root filesystem
/dev/sda2      22222222-2222-2222-2222-222222222222    swap           swap (to be encrypted)
/dev/sda3      33333333-3333-3333-3333-333333333333    /home          home (to be encrypted)
```

In our setting, you will only need to remember a master passphrase to open a home crypto container. After this happens, a system will take passphrases for remaining crypto containers (a sole swap in our setting) from a home just opened. So that remaining passphrases may have any length.

Let's go. First, backup everything into a safe place:
```console
> sudo cp /etc/{fs,crypt}tab /mnt
> sudo mkdir /mnt/home.bak
> cd /home
> sudo find . -depth -print0 | cpio -apmd0 /mnt/home.bak
> sudo umount /home
```

Now replace sda3 device content by crypto container (*this will destroy old unencrypted /home*):
```console
> # create a LUKS crypto container with ext4 file system inside on sda3 device (a passphrase will be asked) and mount it to /home
> sudo /root/bin/fsLuks --build cr_home --data /dev/disk/by-uuid/33333333-3333-3333-3333-333333333333 --mount /home --fstype ext4
```

Then copy the home from backup to crypto container just created:
```console
> cd /mnt/home.bak
> sudo find . -depth -print0 | cpio -apmd0 /home
```

Configure partitions:
```console
> cat /etc/fstab
UUID=11111111-1111-1111-1111-111111111111  /          ext4   acl,user_xattr         1  1
/dev/mapper/cr_home                        /home      ext4   acl,user_xattr,nofail  0  2
/dev/mapper/cr_swap                        swap       swap   defaults,nofail        0  0
tmpfs                                      /tmp       tmpfs  size=5%                0  0
tmpfs                                      /var/tmp   tmpfs  size=5%                0  0

> cat /etc/crypttab
cr_home  UUID=33333333-3333-3333-3333-333333333333  none                      luks
cr_swap  UUID=22222222-2222-2222-2222-222222222222  /home/user/.pki/swap.key  luks,noearly
```
Here, a crypttab configuration for swap (4th column) has no option "swap" to avoid a swap re-creating every time the system boots.

Make the key file for swap crypto container:
```console
> head -c 1024 /dev/random | uuencode -m - | head -n 23 | tail -n 22 > /home/user/.pki/swap.key
> chown 600 /home/user/.pki/swap.key
```

Rewrite a swap in sda2 device by fresh encrypted swap (*this will destroy old unencrypted swap*):
```console
> sudo swapoff -a
> sudo /root/bin/fsLuks --verbose --build cr_swap --data /dev/disk/by-uuid/22222222-2222-2222-2222-222222222222 --mount none --fstype swap --key /home/user/.pki/swap.key
```

Finalize:
```console
> sudo mkinitrd                 # to rebuild initramfs after changing the /etc/crypttab file
> sudo systemctl daemon-reload  # to reparse /etc/fstab and pick up the changes by systemd after changing /etc/{crypt,fs}tab files
> sudo init 1                   # switch host to the single user mode
# rm -rf /tmp/* /var/tmp/*      # prepare temporary directories to re-create as RAM disks
# reboot                        # load new configuration
```

## Originally was published in:
* ```http://www.dvgu.ru/forum/thread.php?threadid=1518&page=2#post43879``` (related post was saved in https://web.archive.org/web/20090130062820/http://www.dvgu.ru:80/forum/thread.php?postid=49410&sid=42448de669099b407199d892c2e38131#post49410 ) 2008-11-07
* https://www.linuxquestions.org/questions/linux-security-4/luks-automation-script-545715/ 2007-13-04
