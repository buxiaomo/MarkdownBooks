# WARNING: No swap limit support

```
vim /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=memory swapaccount=1"

update-grub
reboot
```