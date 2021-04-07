# zram
Sometimes, you don't have enough RAM to feed everything running on your computer, this often happens when you have less than 16 GB of RAM but sometimes even 16 GB isn't enough, depending on your needs. You can use swap partition/file on your drive for that, less used pages of RAM will be sent there when you don't have enough RAM, the problem is that any drive is always way slower than RAM, even the fastest SSD. Here, zram comes into play, zram is basically a swap device inside your RAM, it compresses pages sent to it so that you have more space to store data, at the cost of a bit of extra CPU usage, while still maintaining speed near to RAM speed and improving overall performance compared to a swap on disk. It is recommended to have some swap on disk too, if you're low on RAM, like 4 GB of RAM.



## Setting up GRUB
First of all, you'll need to change GRUB settings to initialize zram at boot. For that, copy & paste that command in your terminal:

`$ sudo nano /etc/default/grub`

Then, change the following line by adding `zram.num_devices=2` at the end, where "2" is the amount of CPU cores you have, that will initialize as much zram devices as your CPU cores, this can lead to a performance boost in multithreading or NUMA.
For example, you will have that line in the end:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash zram.num_devices=2"
```

To exit editor, press "Ctrl + X", then press "Y" and then "Enter".
Now, update your GRUB configuration for changes to apply:

`$ sudo update-grub`

Restart your computer.



## Automatic zram initialization and deinitialization
Create "zram-start.sh" script for automatic zram initialization:

`$ sudo nano /usr/local/bin/zram-start.sh`

And then copy the following code in it while changing variables when said to according to your computer specifications:

```
#!/bin/bash

# Replace 2 with the amount of your CPU cores.
NRDEVICES=2

modprobe zram num_devices=${NRDEVICES}

# Replace 4096 with the amount of your physical RAM in megabytes.
totalmem=4096
mem=$(((totalmem / 2 / ${NRDEVICES}) * 1024 * 1024))

for i in $(seq ${NRDEVICES}); do
  DEVNUMBER=$((i - 1))
  echo zstd > /sys/block/zram${DEVNUMBER}/comp_algorithm
  echo $mem > /sys/block/zram${DEVNUMBER}/disksize
  mkswap /dev/zram${DEVNUMBER}
  swapon -p 75 /dev/zram${DEVNUMBER}
done
```

Change "2" in the `NRDEVICES=2` line according to your CPU cores amount.
Also you need to change "4096" value in the `totalmem=4096` line to the amount of your RAM in megabytes (4096 MB = 4 GB of RAM). This value will then be divided in half and by the amount of your CPU cores and translated to bytes in order to split half of your RAM size for all of your zram devices.
`echo zstd` sets zstd as compression algorithm for zram, it has the best comression ratio which is especially useful in low RAM conditions and also it has the best text compression which is useful while working with loads of text like in a word processor program. Other popular compression algorithms for zram are lzo-rle and lz4, lz4 has the fastest compression and decompression speed and maintains compression ratio near to lzo-rle which has some optimizations but both are inferior in compression ratio compared to zstd. In my usage, neither lzo-rle or lz4 gave any speed boost in real-life situations, neither on a slow more than a decade old PC or on a powerful machine, so zstd is the way to go here since it provides more compression which leads to more space to store data.
`-p 75` sets swap priority of 75 to zram for computer to use it first until all zram devices are full, because disk swap has default priority of -2 and will not be used until zram is full.
Now create "zram-stop.sh" script for automatic zram deinitialization:

`$ sudo nano /usr/local/bin/zram-stop.sh`

And paste following code in it:

```
#!/bin/bash

# Replace 2 with the amount of your CPU cores.
NRDEVICES=2

for i in $(seq ${NRDEVICES}); do
  DEVNUMBER=$((i - 1))
  swapoff /dev/zram${DEVNUMBER}
  echo 1 > /sys/block/zram${DEVNUMBER}/reset
  sleep .5
  modprobe -r zram
done
```

Again change the "2" in the `NRDEVICES=2` line to your amount of CPU cores.
You've successfully created scripts for initialization of zram! A couple more steps to go.



## Set-up systemd service
Firstly, allow execution of the scripts you've just made:

`$ sudo chmod ugo+x /usr/local/bin/zram-start.sh`

`$ sudo chmod ugo+x /usr/local/bin/zram-stop.sh`

Now to automatically launch zram at start-up you need to create systemd unit for it:

`$ sudo systemctl edit --full --force zram.service`

And paste the following in it:

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

Now you need to reload systemd configuration for it to see the changes you've made:

`$ sudo systemctl daemon-reload`

Then start zram service and add it to start-up:

`$ sudo systemctl start zram`

`$ sudo systemctl enable zram`



## Swap optimization
The last thing you need to do, open and edit "sysctl.conf":

`$ sudo nano /etc/sysctl.conf`

Add these lines at the end of the file:

```
vm.vfs_cache_pressure=500
vm.swappiness=100
vm.dirty_background_ratio=1
vm.dirty_ratio=50
vm.page-cluster=0
```

`vm.vfs_cache_pressure=500` tells system to store less cache in your RAM.

`vm.swappiness=100` sets swappiness to 100 for your system to start using zram earlier.

`vm.dirty_background_ratio=1` and `vm.dirty_ratio=50` controls the flow of "dirty" pages to swap.

`vm.page-cluster=0` controls the number of pages up to which consecutive pages are read in from swap in a single attempt. Page cluster of 0 limits that to 1 page per attempt, which arrives lower latency.



# Finish line
Just to be sure, I recommend rebooting your computer at that point. Your zram should be up and running.
To check that, you can run "zramctl":

`$ zramctl`

Example output could be:

```
NAME       ALGORITHM DISKSIZE   DATA COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram1 zstd            1G 144.9M 24.7M 31.4M       2 [SWAP]
/dev/zram0 zstd            1G 146.5M   25M 31.4M       2 [SWAP]
```
