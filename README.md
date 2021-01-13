# lxc_gui
X11 software compatible LXC containers for Arducopter Drone Simulations. The following lxc gui process has been taken from [various](https://blog.simos.info/how-to-easily-run-graphics-accelerated-gui-apps-in-lxd-containers-on-your-ubuntu-desktop/) [blog](https://blog.simos.info/how-to-run-graphics-accelerated-gui-apps-in-lxd-containers-on-your-ubuntu-desktop/) [posts](https://blog.simos.info/running-x11-software-in-lxd-containers/) by [Simos Xenitellis](https://blog.simos.info/author/simos/) and combined here.


# Environement Setup
## Initial Configuration 
(Done once on Host)

### Installing LXD
To install LXD, see the website for instructions. For Ubuntu 16.04 and later, use the command:
```
$ sudo apt-get install lxd 
```
Run the following to verify which LXD channel you are tracking:
```
$ snap info lxd 
```
Now, refresh the LXD snap package into the 'latest/candidate' channel: 
```
$ sudo snap refresh lxd --channel=latest/candidate 
lxd (candidate) 4.6 from Canonical refreshed
$
```
NOTE: If you have setup the latest/candidate channel, you should be now switch to the latest/stable channel. LXD 4.6 has been released now into the stable channel. Use the following command:
```
$ sudo snap refresh lxd --channel=latest/stable 
```
In the following on the host (only once) the following command (Note: if you do not have the bash shell, then \$UID is the user-id of the current user. You can replace \$UID with \$(id -u) in that case.) To appends a new entry in both the /etc/subuid and /etc/subgid subordinate UID/GID files. It allows the LXD service (runs as root) to remap our user’s ID (\$UID, from the host) as requested.
```
$ echo "root:$UID:1" | sudo tee -a /etc/subuid /etc/subgid
[sudo] password for myusername: 
root:1000:1
$ 
```
If this is your first time using LXD on your machine, then you will need to create a storage pool. This is done with the command:
```
lxd init 
```
You will be prompted many things, you can just hit enter for all of them to use the default settings. 

### Configuring Audio (host)

The audio server in Ubuntu desktop is Pulseaudio, and Pulseaudio has a feature to allow authenticated access over the network. We install the paprefs (PulseAudio Preferences) package on the host.
```
$ sudo apt install paprefs
...
$ paprefs
```
Under the 'Network Server' options, enable the following:
- Enable network access to local sound devices
    - Allow other machines on the LAN to discover local sound devices
    - don't require authentication

Under the 'Multicast/RTP' options, enable the following:
- Enable Multicast/RTP receiver

All of the other options can remain unchecked.

### LXD GUI Profile
Now we will make an LXD profile that will help automatically setup a LXD container to run X11 applications on the host's X server. Run the command:
```
$ echo $DISPLAY
:1
```
In my case, the output was ':1', so for the following code I use the value **X1** (as already shown below). Your output may be ':0', in which case you should use X0 instead. **If your host computer is not using nvidiagraphics, then set nvidia.runtime: "false"**. Copy the following text and save it to a file named **x11.profile**. 
```
config:
  environment.DISPLAY: :0
  environment.PULSE_SERVER: unix:/home/ubuntu/pulse-native
  nvidia.driver.capabilities: all
  nvidia.runtime: "true"
  user.user-data: |
    #cloud-config
    runcmd:
      - 'sed -i "s/; enable-shm = yes/enable-shm = no/g" /etc/pulse/client.conf'
    packages:
      - x11-apps
      - mesa-utils
      - alsa-utils
      - pulseaudio
      - paprefs
description: GUI LXD profile
devices:
  PASocket1:
    bind: container
    connect: unix:/run/user/1000/pulse/native
    listen: unix:/home/ubuntu/pulse-native
    security.gid: "1000"
    security.uid: "1000"
    uid: "1000"
    gid: "1000"
    mode: "0777"
    type: proxy
  X0:
    bind: container
    connect: unix:@/tmp/.X11-unix/X1
    listen: unix:@/tmp/.X11-unix/X0
    security.gid: "1000"
    security.uid: "1000"
    type: proxy
  mygpu:
    type: gpu
name: x11
used_by: []

```

Then, create the profile with the following commands. This creates a profile called **x11**:
```
$ lxc profile create x11
Profile x11 created
$ cat x11.profile | lxc profile edit x11
$   
```

## Container Configuration 
(Done for Each New Container)
### Generate Container
To create a container, run the following. In this case, I am creating a container from the default LXD ubuntu 18.04 to be named 'mycontainer'.
```
$ lxc launch ubuntu:18.04 --profile default --profile x11 mycontainer
```
To get a shell in the container, run the following.
```
$ lxc exec mycontainer -- sudo --user ubuntu --login
```
Once we get a shell inside the container, install updates: 
```
ubuntu@mycontainer:~$ sudo apt update
ubuntu@mycontainer:~$ sudo apt upgrade
```

### Configuring Audio (container)
First, the IP address of the host (that has Pulseaudio) is the IP of the lxdbr0 interface, or the default gateway (ip link show). Second, the authorization is provided through the cookie in the host at /home/${USER}/.config/pulse/cookie Let’s connect these to files inside the container.
```
lxc exec mycontainer -- sudo --login --user ubuntu
ubuntu@mycontainer:~$ echo export PULSE_SERVER="tcp:`ip route show 0/0 | awk '{print $3}'`" >> ~/.profile
```
This command will automatically set the variable PULSE\_SERVER to a value like tcp:10.0.185.1, which is the IP address of the host, for the lxdbr0 interface. The next time we log in to the container, PULSE\_SERVER will be configured properly. By default, the Pulseaudio cookie is found at ~/.config/pulse/cookie. The directory tree ~/.config/pulse/ does not exist, and if we do not create it ourselves, then lxd config will autocreate it with the wrong ownership. So, we create it (mkdir -p), then add the correct PULSE\_COOKIE line in the configuration file ~/.profile. Finally, we exit from the container and mount-bind the cookie from the host to the container. When we log in to the container again, the cookie variable will be correctly set.
```
ubuntu@mycontainer:~$ mkdir -p ~/.config/pulse/
ubuntu@mycontainer:~$ echo export PULSE_COOKIE=/home/ubuntu/.config/pulse/cookie >> ~/.profile
ubuntu@mycontainer:~$ exit
$ lxc config device add mycontainer PACookie disk path=/home/ubuntu/.config/pulse/cookie source=/home/${USER}/.config/pulse/cookie
```
Everything should now be configured. Log into the container and run the diagnostic commands:
```
ubuntu@mycontainer:~$ glxinfo -B
...
ubuntu@mycontainer:~$ nvidia-smi 
...
ubuntu@mycontainer:~$ pactl info
...
```
You can run `xclock` which is an Xlib application. If it runs, it means that unaccelerated (standard X11) applications are able to run successfully.

You can run `glxgears` which requires OpenGL. If it runs, it means that you can run GPU accelerated software.

To test the audio you can run the following command. You can open a second terminal and run it at  the same time on your host computer to confirm that both the host and the container are able to output audio: `speaker-test -Dpulse -c6 -twav`

# Gazebo Drone Simulation Container Setup

## Generate and Configure Container
The first step is to clone this repository:
```
$ git clone https://github.com/buczek-j/lxc_gui.git
```
Next, download the [container tar-ball](https://drive.google.com/file/d/1mxkuoqp06grlAz8yqaszPbyLC07xBybQ/view?usp=sharing) to your local images 
```
$ mkdir /home/localuser/myimages/
$ cp ~/Downloads/DroneSim-U18-04.tar.bz /home/localuser/myimages/DroneSim-U18-04.tar.bz
```
Import the Gazebo Image from the tar-ball
```
$ lxc image import  /home/localuser/myimages/DroneSim-U18-04.tar.bz --alias Drone-Sim-Image
```
After following the steps above for setting up the Profile x11 (shown above), generate the container with the following command:
```
$ lxc launch Drone-Sim-Image --profile default --profile x11 drone-sim-container
$ lxc exec drone-sim-container -- sudo --user ubuntu --login
```

**Now follow the Conatiner Configuration setup listed above**

## Running Drone Simulation

Running the drone simulation is simple. All you need to do is to log into the the container with one terminal and run the shell script:
```
ubuntu@drone-sim-container:~$ . ~/launch_gazebo.sh 1
```
**NOTE: The simulator also creates pop-up terminals. Do not close any these terminals, it will interupt the simulation**

Where the number afterwards is the number of drones to simulate (above we generate 1 drone). The shell script generates gazebo world files from an erb world file and erb model. So any number of drones could be generated, the only limitations would be computational power and port allocation. For reference, this simulator has successfully tested 50 vehicles. 

When you are done with the simulation, close everything by typing `Ctrl + C` in the container terminal. This will close the gazebo simulation as well as all of the pop-up terminals. 

The SITL for the drones opens a TCP socket on the External IP of the container. This means that any flight control program (i.e. [BasicArdu](https://github.com/buczek-j/BasicArducopter)) can be run on the host machine and while the gazebo simulation is run on the container. The IP of the container gets displayed in the terminal where the `launch_gazebo.sh` was run. For example, the connection string might be "tcp:10.91.238.136:5762". For multiple vehicles, the ip remains the same, but the port number will increase by 10, for example:
- "tcp:10.91.238.136:5762" for drone_0 (first drone instance)
- "tcp:10.91.238.136:5772" for drone_1
- "tcp:10.91.238.136:5782" for drone_2
- . . .
