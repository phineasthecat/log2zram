# Log2Zram

Usefull for IoT / maker projects for reducing SD, Nand and Emmc block wear via log operations.
Uses Zram to minimise precious memory footprint and extremely infrequent write outs.

Log2Zam is a lower write fork https://github.com/azlux/log2ram based on transient log for Systemd here : [A transient /var/log](https://www.debian-administration.org/article/661/A_transient_/var/log)
Many thanks to Azlux for the great initial project.

Can not be used for mission critical logging applications where a system crash and log loss is unaceptable.
If the extremely unlikely event of a system crash is not a major concern then L2Z can massively reduce log block wear whilst maintaining and extremely tiny memory footprint. Further resilience can be added by the use of a watchdog routine to force stop.

Works well in conjunction with https://github.com/StuartIanNaylor/zramdrive and https://github.com/StuartIanNaylor/zram-swap-config
_____
## Menu
1. [Install](#install)
2. [Config](#config)
3. [It is working ?](#it-is-working)
4. [Uninstall](#uninstall-)

## Install
    sudo apt-get install git
    git clone https://github.com/StuartIanNaylor/log2zram
    cd log2zram
    sudo sh install.sh
    

## Customize
#### variables :
In the file `/etc/log2zram.conf` sudo nano /etc/log2zram.conf to edit:
```
# Size for the zram memory used, it defines the mem_limit for the zram device.
# The default is 20M and is basically enough for minimal applications.
# Because aplications can often vary in logging frequency this may have to be tweaked to suit application .
SIZE=20M
# COMP_ALG this is any compression algorithm listed in /proc/crypto
# lz4 is fastest with lightest load but deflate (zlib) and Zstandard (zstd) give far better compression ratios
# lzo is very close to lz4 and may with some binaries have better optimisation
# COMP_ALG=lz4 for speed or deflate for compression, lzo or zlib if optimisation or availabilty is a problem
COMP_ALG=lz4
# LOG_DISK_SIZE is the uncompressed disk size. Note zram uses about 0.1% of the size of the disk when not in use
# LOG_DISK_SIZE is expected compression ratio of alg chosen multiplied by log SIZE where 300% is an approx good level.
# lzo/lz4=2.1:1 compression ratio zlib=2.7:1 zstandard=2.9:1
# Really a guestimate of a bit bigger than compression ratio whilst minimising 0.1% mem usage of disk size
LOG_DISK_SIZE=60M

# ****************** Scheduler settings for logrotate override and prune frequencies **********************
# LOGROTATE_FREQ & PRUNE_FREQ are the count in minutes each operation will take place
# LOGROTATE_FREQ= Leave empty to turn off and use normal cron daily or forced logrotate will take place
# LOGROTATE_FREQ=minutes
LOGROTATE_FREQ=
# PRUNE_FREQ will check if available space % is less than PRUNE_LEVEL and if so move and clean /oldlog
# PRUNE_FREQ= Leave empty to disable 
# PRUNE_FREQ=minutes
PRUNE_FREQ=60
# PRUNE_LEVEL if log size is below this level then old logs will be moved to hdd.log enter as % of free space
# Moving the old logs will restart log rotation as old logs will no longer exist in /var/log/oldlog
# In normal operation hitting 50% or above can take many hourly cycles so a higher prune level is a balance
# 20-40% is probably a good level as too high will restart logrotation and create less history, too low and increase
# the chance of running out of space
PRUNE_LEVEL=30
```

#### refresh time:
By default Log2Zram checks available log space every hour (PRUNE_FREQ=60). It them makes a comparison of the available space percentage against Prune_Level and only writes out old logs to disk when triggered (if lower) and then removes the collected old logs from zram space. PRUNE_FREQ= Leave empty to disable
For low space considerations you can also increase the daily logrotate by setting LOGROTATE_FREQ=360 for 4 times daily if left as LOGROTATE_FREQ= then this function remains off and normal daily cron Logrotate will function or forced logrotate will take place

### It is working?
```
pi@raspberrypi:~/log2zram $ df "/var/log" -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/zram0       55M  2.6M   48M   6% /var/log
…
pi@raspberrypi:~/log2zram $ zramctl
NAME       ALGORITHM DISKSIZE  DATA  COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram0 lz4            60M  6.7M 903.5K  1.2M       1 /var/log
```

### Testing
Force write out any updated logs to persistant HDD Dir. Useful if new app install/changes rather than start/stop or reboot
```
sudo sh /usr/local/bin/log2zram/log2zram write
```
Force prune of oldlog (copy to prune.log then delete oldlog contents). When tuning start log2zram with clean oldlog
```
sudo sh /usr/local/bin/log2zram/log2zram prune
```
Force verbose logrotate
```
sudo logrotate -vf /etc/logrotate.conf
```



| Compressor name	     | Ratio	| Compression | Decompress. |
|------------------------|----------|-------------|-------------|
|zstd 1.3.4 -1	         | 2.877	| 470 MB/s	  | 1380 MB/s   |
|zlib 1.2.11 -1	         | 2.743    | 110 MB/s    | 400 MB/s    |
|brotli 1.0.2 -0	     | 2.701	| 410 MB/s	  | 430 MB/s    |
|quicklz 1.5.0 -1	     | 2.238	| 550 MB/s	  | 710 MB/s    |
|lzo1x 2.09 -1	         | 2.108	| 650 MB/s	  | 830 MB/s    |
|lz4 1.8.1	             | 2.101    | 750 MB/s    | 3700 MB/s   |
|snappy 1.1.4	         | 2.091	| 530 MB/s	  | 1800 MB/s   |
|lzf 3.6 -1	             | 2.077	| 400 MB/s	  | 860 MB/s    |

Zstd & Zlib are great for text compression where ratios of up to 3.3 can be obtained. Generally 230% for LZO/LZ4 LOG_DISK_SIZE over the Mem_Limit of Size should be OK, with Zlib/Zstd maybe even up to 350% can be expected.
Most logrotate schedules compress after the second stored log (log.1) and the ratio between uncompressed and compressed log can have much effect. So it dependent on your logging setup with LZO/LZ4 ranging from 210-250% ZSTD/ZLIB 290-350% for the same Mem_Limit Size.
Generally these are minimum compression rates but how much whitespace and zeroes does a file contain? 

## Uninstall
```
sudo sh /usr/local/bin/log2zram/uninstall.sh
```
/var/hdd.log /var/prune.log is retained on uninstall and only removed on new install.

## Git Branches & Update
From the command line, enter `cd <path_to_local_repo>` so that you can enter commands for your repository.
Enter `git add --all` at the command line to add the files or changes to the repository
Enter `git commit -m '<commit_message>'` at the command line to commit new files/changes to the local repository. For the <commit_message> , you can enter anything that describes the changes you are committing.
Enter `git push`  at the command line to copy your files from your local repository to remote.
