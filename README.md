# zRAM
zRAM is a very neat and useful feature of Linux, it is a compressed swap right in your RAM. Because it's compressed, same amount of data takes up less space, so you can store more, for example 1024 MB of data sended to RAM can take up only less than 512 MB in zRAM, which gives you ability to store more data with the same amount of physical RAM. And because that swap is stored directly in your RAM, performance is near to the speed of RAM, but not the same because you spend a little bit of CPU usage to compress/decompress data in zRAM, and still it is way faster than a swap on any SSD or HDD.
It is recommended to set-up zRAM to take up 50% of your physical RAM in total.



## Prerequisites
* Linux kernel should be at least version 5.1.
* GRUB boot loader.



## Setting up GRUB
First of all, change GRUB settings, for this open `/etc/default/grub` as root.

`$ sudo nano /etc/default/grub`

Then, change line `GRUB_CMDLINE_LINUX_DEFAULT="..."` by adding `zram.num_devices=1` after a space at the end of inside quotation marks. For example result line could look like this:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zram.num_devices=1"
```

Then, update GRUB configuration.

`$ sudo update-grub`

Now, restart your computer.



## Automatic zRAM initialization and and stop script
Create a script "zram-start.sh" in /usr/local/bin/.

`$ sudo nano /usr/local/bin/zram-start.sh`

Copy the following code in it:

```
#!/bin/bash

modprobe zram num_devices=1

# Replace 4096 with the amount of your physical RAM in megabytes.
totalmem=4096
mem=$(((totalmem / 2 / ${NRDEVICES}) * 1024 * 1024))

echo $mem > /sys/block/zram0/disksize
mkswap /dev/zram0
swapon -p 75 /dev/zram0
```

Change 4096 in line `totalmem=4096` to the size of your RAM in megabytes, as said in the comment. Now create zram-stop.sh with the following code in it.

`$ sudo nano /usr/local/bin/zram-stop.sh`

```
#!/bin/bash

swapoff /dev/zram0
echo 1 > /sys/block/zram0/reset
sleep .5
modprobe -r zram
```

Allow execution of those scripts.

`$ sudo chmod ugo+x /usr/local/bin/zram-start.sh`

`$ sudo chmod ugo+x /usr/local/bin/zram-stop.sh`



## Set-up systemd service
To automatically launch zRAM at start-up you need to create systemd unit for it.

`$ sudo systemctl edit --full --force zram.service`

```
[Unit]
Description=zRAM block devices swapping
[Service]
Type=oneshot
ExecStart=/usr/local/bin/zram-start.sh
ExecStop=/usr/local/bin/zram-stop.sh
RemainAfterExit=true
[Install]
WantedBy=multi-user.target
```

Now reload systemd configuration.

`$ sudo systemctl daemon-reload`

Start zRAM service and add it to start-up.

`$ sudo systemctl start zram`

`$ sudo systemctl enable zram`



## Swap optimization
The last thing you need to do. Open and edit sysctl.conf.

`$ sudo nano /etc/sysctl.conf`

Add two lines at the end of the file.

```
vm.swappiness = 80
vm.page-cluster = 0
```

`vm.swappiness = 80` sets swappiness to 80 rather than default 60 because zRAM is fast and we can use it more often than the disk swap, higher swappiness means more often swap usage, so you could benefit from higher swappiness because you use zRAM.
`vm.page-cluster = 0` controls the number of pages up to which consecutive pages are read in from swap in a single attempt. Page cluster of 0 sets that to 1 page per attempt, which arrives less latency and zRAM could benefit from it.



# zRAM is ready
Just to be sure, I recommend rebooting your computer at that point. Your zRAM should be up and running, you can check that via `$ zramctl` and `$ swapon`
