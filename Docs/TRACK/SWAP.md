# Swap Optimization
*The below documentation has been lifted in its entierty from a [Digital Ocean tutorial](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04). All credit goes to the original author [Brian Boucheron](https://www.digitalocean.com/community/users/bboucheron).*

*It is included here for the purposes of consolidating this Traccar tutorial all in one place.*

## What is Swap?
Swap is a portion of hard drive storage that has been set aside for the operating system to temporarily store data that it can no longer hold in RAM. This lets you increase the amount of information that your server can keep in its working memory, with some caveats. The swap space on the hard drive will be used mainly when there is no longer sufficient space in RAM to hold in-use application data.

The information written to disk will be significantly slower than information kept in RAM, but the operating system will prefer to keep running application data in memory and use swap for the older data. Overall, having swap space as a fallback for when your system’s RAM is depleted can be a good safety net against out-of-memory exceptions on systems with non-SSD storage available.

## Step 1 – Checking the System for Swap Information

Before we begin, we can check if the system already has some swap space available. It is possible to have multiple swap files or swap partitions, but generally one should be enough. We can see if the system has any configured swap by typing:\
`$ sudo swapon --show`

If you don’t get back any output, this means your system does not have swap space available currently. You can verify that there is no active swap using the free utility:\
`$ free -h`
```
Output
              total        used        free      shared  buff/cache   available
Mem:          981Mi       122Mi       647Mi       0.0Ki       211Mi       714Mi
Swap:            0B          0B          0B
```
As you can see in the Swap row of the output, no swap is active on the system.

## Step 2 – Checking Available Space on the Hard Drive Partition

Before we create our swap file, we’ll check our current disk usage to make sure we have enough space. Do this by entering:\
`$ df -h`

```
Output
Filesystem      Size  Used Avail Use% Mounted on
udev            474M     0  474M   0% /dev
tmpfs            99M  932K   98M   1% /run
/dev/vda1        25G  1.4G   23G   7% /
tmpfs           491M     0  491M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           491M     0  491M   0% /sys/fs/cgroup
/dev/vda15      105M  3.9M  101M   4% /boot/efi
/dev/loop0       55M   55M     0 100% /snap/core18/1705
/dev/loop1       69M   69M     0 100% /snap/lxd/14804
/dev/loop2       28M   28M     0 100% /snap/snapd/7264
tmpfs            99M     0   99M   0% /run/user/1000
```

The device with `/` in the `Mounted on` column is our disk in this case. We have plenty of space available in this example (only 1.4G used). Your usage will probably be different.

Although there are many opinions about the appropriate size of a swap space, it really depends on your personal preferences and your application requirements. Generally, an amount equal to or double the amount of RAM on your system is a good starting point. Another good rule of thumb is that anything over 4G of swap is probably unnecessary if you are just using it as a RAM fallback.

## Step 3 – Creating a Swap File

Now that we know our available hard drive space, we can create a swap file on our filesystem. We will allocate a file of the size that we want called `swapfile` in our root (`/`) directory.

The best way of creating a swap file is with the `fallocate` program. This command instantly creates a file of the specified size.

Since the server in our example has 1G of RAM, we will create a 1G file in this guide. Adjust this to meet the needs of your own server:\
`$ sudo fallocate -l 1G /swapfile`

We can verify that the correct amount of space was reserved by typing:\
`$ ls -lh /swapfile`\
Which returns:
```
-rw-r--r-- 1 root root 1.0G Apr 25 11:14 /swapfile
```
Our file has been created with the correct amount of space set aside.

## Step 4 – Enabling the Swap File

Now that we have a file of the correct size available, we need to actually turn this into swap space.

First, we need to lock down the permissions of the file so that only users with root privileges can read the contents. This prevents normal users from being able to access the file, which would have significant security implications.

Make the file only accessible to root by typing:\
`$ sudo chmod 600 /swapfile`

Verify the permissions change by typing:\
`$ ls -lh /swapfile`\
Which outputs:
```
-rw------- 1 root root 1.0G Apr 25 11:14 /swapfile
```

As you can see, only the root user has the read and write flags enabled.

We can now mark the file as swap space by typing:\
`$ sudo mkswap /swapfile`\
Which outputs:
```
Setting up swapspace version 1, size = 1024 MiB (1073737728 bytes)
no label, UUID=6e965805-2ab9-450f-aed6-577e74089dbf
```

After marking the file, we can enable the swap file, allowing our system to start using it:\
`$ sudo swapon /swapfile`

Verify that the swap is available by typing:\
`$ sudo swapon --show`\
Output:
```
NAME      TYPE  SIZE USED PRIO
/swapfile file 1024M   0B   -2
```

We can check the output of the `free` utility again to corroborate our findings:\
`$ free -h`\
Output:
```
              total        used        free      shared  buff/cache   available
Mem:          981Mi       123Mi       644Mi       0.0Ki       213Mi       714Mi
Swap:         1.0Gi          0B       1.0Gi
```

Our swap has been set up successfully and our operating system will begin to use it as necessary.

## Step 5 – Making the Swap File Permanent

Our recent changes have enabled the swap file for the current session. However, if we reboot, the server will not retain the swap settings automatically. We can change this by adding the swap file to our `/etc/fstab` file.

Back up the `/etc/fstab` file in case anything goes wrong:\
`$ sudo cp /etc/fstab /etc/fstab.bak`

Add the swap file information to the end of your /etc/fstab file by typing:\
`$ echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab`

Next we’ll review some settings we can update to tune our swap space.

## Step 6 – Tuning your Swap Settings

There are a few options that you can configure that will have an impact on your system’s performance when dealing with swap.

### Adjusting the Swappiness Property
The `swappiness` parameter configures how often your system swaps data out of RAM to the swap space. This is a value between 0 and 100 that represents a percentage.

With values close to zero, the kernel will not swap data to the disk unless absolutely necessary. Remember, interactions with the swap file are “expensive” in that they take a lot longer than interactions with RAM and they can cause a significant reduction in performance. Telling the system not to rely on the swap much will generally make your system faster.

Values that are closer to 100 will try to put more data into swap in an effort to keep more RAM space free. Depending on your applications’ memory profile or what you are using your server for, this might be better in some cases.

We can see the current swappiness value by typing:\
`$ cat /proc/sys/vm/swappiness`\
Output:
```
60
```

For a Desktop, a swappiness setting of 60 is not a bad value. For a server, you might want to move it closer to 0.

We can set the swappiness to a different value by using the `sysctl` command.

*Note here from WMTaylor3, not part of the original article. I went for a setting of 30 in order to ensure our 1GB of memory wouldn't cause us to crash out.*\
For instance, to set the swappiness to 10, we could type:\
`$ sudo sysctl vm.swappiness=10`\
Output:
```
vm.swappiness = 10
```

This setting will persist until the next reboot. We can set this value automatically at restart by adding the line to our /etc/sysctl.conf file:\
`$ sudo nano /etc/sysctl.conf`

At the bottom, you can add:\
`vm.swappiness=10`

Save and close the file when you are finished.

### Adjusting the Cache Pressure Setting
Another related value that you might want to modify is the `vfs_cache_pressure`. This setting configures how much the system will choose to cache inode and dentry information over other data.

Basically, this is access data about the filesystem. This is generally very costly to look up and very frequently requested, so it’s an excellent thing for your system to cache. You can see the current value by querying the `proc` filesystem again:\
`$ cat /proc/sys/vm/vfs_cache_pressure`\
Output:
```
100
```

As it is currently configured, our system removes inode information from the cache too quickly. We can set this to a more conservative setting like 50 by typing:\
`$ sudo sysctl vm.vfs_cache_pressure=50`\
Output:
```
vm.vfs_cache_pressure = 50
```

Again, this is only valid for our current session. We can change that by adding it to our configuration file like we did with our swappiness setting:\
`$ sudo nano /etc/sysctl.conf`

At the bottom, add the line that specifies your new value:\
`vm.vfs_cache_pressure=50`

Save and close the file when you are finished.

## Conclusion
Following the steps in this guide will give you some breathing room in cases that would otherwise lead to out-of-memory exceptions. Swap space can be incredibly useful in avoiding some of these common problems.

If you are running into OOM (out of memory) errors, or if you find that your system is unable to use the applications you need, the best solution is to optimize your application configurations or upgrade your server.