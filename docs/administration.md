> How to create an account, who has sudo, soft- and hardware setups, ...
---

# Administration

## Some of administrator responsibilities

* All servers must run Ubuntu release that is a) supported b) LTS (long-term). Best, if it is the latest LTS.
* CUDA should support the latest version of PyTorch.
* When adding a new user, the administrator needs to add them to `hf_cache_users` group.

## User management

In general, all user accounts should have the name following the template `<first letter of first name><last name>`.
For example, for a user Alexey Romanov, the username would be `aromanov`.

When creating a user do the following

```bash
NAME=<username>
sudo adduser $NAME  # create a user
sudo passwd -e $NAME  # require the user to change their password on the first login
sudo usermod -a -G hf_cache_users $NAME  # add the user to the group that has access to the Huggingface cache
sudo chmod -R 2770 /home/hf_cache  # not entirely sure why we need this, but we do
```
    
> Remember to add users to the group `hf_cache_users`!

## Turn off GUI on servers

GUI (xorg process) in Ubuntu 20.04 can eat up to 300Mb of GPU RAM, which is **A LOT**. To disable this process follow [this guide](https://askubuntu.com/questions/16371/how-do-i-disable-x-at-boot-time-so-that-the-system-boots-in-text-mode).

## HuggingFace Cache management

> Cache management might be trickly. Please make sure you understand all of the commands below, especially `setfacl` and `g+s`
> Read more about setfacl and `g+s` [in this stackoverflow](https://stackoverflow.com/questions/31886206/linux-set-user-and-group-ownership-for-future-files-and-folders)

Each text machine server has a common huggingface cache directory. Sharing models and datasts across users saves hundreds of gigabytes of space.
This is implemented via environment variables `TRANSFORMERS_CACHE` and `HF_DATASETS_CACHE` that are set in `/etc/environment`.

```bash
TRANSFORMERS_CACHE="/home/hf_cache/transformers_cache"
HF_DATASETS_CACHE="/home/hf_cache/datasets_cache"
```

> The location of `hf_cache` might be different on different servers. We try to put it on the largerst drive.

The directory `hf_cache` should have the group `hf_cache_users` assigned. This is how we set it up:

```bash
sudo groupadd hf_cache_users  # create the group
sudo chgrp -R hf_cache_users hf_cache  # assign the directory to the group
sudo chmod -R 2770 hf_cache  # allow all group users to read, write and execute files in it and restrict other users from acessing

# VERY IMPORTANT
# set future file and directory permissions to rwx for hf_cache users and --- for others
sudo setfacl -R -d -m group:hf_cache_users:rwx,other::--- hf_cache
sudo setfacl -R -m group:hf_cache_users:rwx,other::--- hf_cache

for user in user names separated by space;
    do sudo usermod -a -G hf_cache_users $user;  # add users to the group
    done
```

## Installing drivers and CUDA

### Ubuntu >= 20.04

**EXTREMELY IMPORTANT: Use tmux** or screen to run the installation or you can corrupt the whole server if you accidentally close the terminal or lose connection.

1. Remove stuff installed via apt-get
    ```bash
    sudo apt-get purge cuda
    sudo apt-get purge nvidia-cuda-toolkit
    sudo apt-get purge "cuda*"
    sudo apt autoremove
    ```
2. Go to [Nvidia website](https://developer.nvidia.com/cuda-downloads) select Linux, x86, Ubuntu, 20.04, deb (network). It will show you commands like these, execute them.
```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-keyring_1.0-1_all.deb
sudo dpkg -i cuda-keyring_1.0-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda
```
3. Reload the computer `sudo reboot`
4. Follow official Nvidia tutorial to install CUDNN. To download cudnn directly to the server using wget use [this hack](https://stackoverflow.com/questions/31279494/how-to-install-cudnn-from-command-line)
5. Check that `torch.cuda.is_available()` **and** that something like `torch.zeros(10).cuda()` works. Fix all warnings or errors if they appear.

### CUDA installation errors
1. If you get `The following signatures couldn't be verified`, follow this to replace GPC keys https://developer.nvidia.com/blog/updating-the-cuda-linux-gpg-repository-key/
2. `unmet dependencies` is a very tricky error, but it probably means that you have some conflicting nvidia packages. Removing all of them before installing cuda might help
```bash
apt clean
apt update
apt purge cuda
apt purge "nvidia-*"
apt autoremove
```

## Useful commands

* View all users: `getent passwd`
* View sudo users: `getent group sudo | cut -d: -f4`
* Delete user from sudo: `sudo deluser USERNAME sudo`
* Add user to sudo: `sudo usermod -aG sudo USERNAME`
* Add user: `sudo adduser USERNAME`
* Add a group: `sudo addgroup GROUPNAME`
* Set primary group: `usermod -g GROUPNAME USERNAME`
* Expire password: `sudo passwd -e USERNAME`


## System Monitoring
We use [Netdata](https://www.netdata.cloud) to monitor our servers. Please ask Vlad or Anton to add you to our netdata account.

> Remember to restart Netdata every time you change the config. This is how you do this: `sudo systemctl restart netdata`

If you need to install Netdata to a new server go to our account and click on Nodes -> Add Nodes. It will give you a command to execute on the sever that will install everything and connect it to our account. The command looks roughtly like this:

```bash
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh --claim-token OUR_TOKEN_DONT_SHARE_IT --claim-url https://app.netdata.cloud
```

Next step is to [activate GPU monitoring](https://learn.netdata.cloud/docs/agent/collectors/python.d.plugin/nvidia_smi/). To do this execute
```
cd /opt/netdata/etc/netdata
sudo ./edit-config python.d.conf
```

Remove the comment `#` from the line `nvidia-smi: Yes`.

Then add GPU temperature alerts. Execute `sudo ./edit-config health.d/gpu0_temperature.conf` and pase the following there
```alarm: gpu_0_temperature
on: nvidia_smi.gpu0_temperature
lookup: average -5s
units: C
every: 10s
crit: $this > 80
info: High GPU temperature is potentially dangerous. Turn off the server immediately.
```

Repeat this for each GPU changing `gpu0_temperature -> gpu1_temperature` in file name, in alarm name, and **most important** in `on` parameter.

Now delete the swap alarm, because it is annoying and is not useful for us

```
sudo rm /opt/netdata/etc/netdata/orig/health.d/swap.conf
```

Next step is to [activate Slack notifications](https://learn.netdata.cloud/docs/agent/health/notifications/slack). Go to [Slack Incoming Webhooks configuration](https://text-machine-test.slack.com/services/B046A6A11C2) and copy Webhook URL. In terminal execute `sudo ./edit-config health_alarm_notify.conf` and change `SLACK_WEBHOOK_URL` to this value. Then set `DEFAULT_RECIPIENT_SLACK="hardware"`.

**Finally, restart Netdata** with command `sudo systemctl restart netdata`
