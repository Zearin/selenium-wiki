# Introduction

We have a very nice, dedicated 8 core XServe for our build system with Atlassian Bamboo installed. It is available at:

http://xserve.openqa.org:8085/

Several VMs are also on this machine, allowing for cross-browser and cross-operating system testing. Currently, we have the following build agents/VMs set up:

Running:
  * OS X 10.5 Server (headless on the host server)
  * OS X 10.5 Server (as a VM, **not** headless)
  * Windows XP
  * Ubuntu

Not Running:
  * Windows Vista

Planned:
  * Windows 7

# SSH Settings

If you wish to be able to access the VMs, you'll need direct SSH access to the host server. Ask PatrickLightbody or Adam Goucher for an account, which you can use to SSH into `xserve.openqa.org`. We recommend the following entry in your `.ssh/config`, as it'll make connecting to the VMs much easier:

```
Host qax
	Hostname xserve.openqa.org
	# xp-vm-1 Remote Desktop
	LocalForward 3399 192.168.25.128:3389
	# osx-vm-1 SSH (port 1122) and VNC (display port 10)
	LocalForward 1122 192.168.25.129:22
	LocalForward 5910 192.168.25.129:5900
	# ubuntu-vm-1 SSH (port 1123) and VNC (display port 11)
	LocalForward 1123 192.168.25.132:22
	LocalForward 5911 192.168.25.132:5901
	# vista-vm-1 Remote Desktop (port 3400)
	LocalForward 3400 192.168.25.130:3389
```

If you need to access the main XServe UI (for instance, to play with VMWare) you will want to add this as well. Most do not need to do this though.

```
	# main
	LocalForward 5900 localhost:5900
```

# Host Details

Bamboo is running on the host server as the user **`admin`**. Once SSH'd in to your personal account (using the above SSH config, ideally), you can switch over to this account with the following command:

```sh
sudo su - admin
```

To control bamboo itself you use the **`/Applications/Bamboo/bamboo.sh`** script which is a standard sys v one so start, stop, etc. are all supported.

There is another important user to be aware of: **`admin`**. This user's password is stored locally on the server at `/var/root/ADMIN\_PASSWORD` and can be read by switching to root from your user account (ie: `sudo su -` ) and then reading that file.

Armed with that password, if you need to interact with the actual OS X desktop, you can do so by VNC'ing in to xserve.openqa.org with the username **`admin`** and that password.

Once restarted, you need to also manually [`re`](re.md)connect all the agents to the server.

# VM Details

More often, you'll want to log directly in to the VMs to perform maintenance on them. Using the SSH configuration supplied above, you connect to the VMs like so:

  * Windows XP VM
    * Remote Desktop to `localhost:3399` with the username **`Administrator`** and the standard OpenQA password.
  * Windows Vista
    * Remote Desktop to `localhost:3400` with the username **`openqa`** and the standard OpenQA password.
  * OS X 10.5 Server
    * VNC to `5910` (display port `10`) with username **`openqa`** and the standard OpenQA password.
    * SSH to `localhost` port `1122` with the username **`openqa`** and the standard OpenQA password.
  * Ubuntu
    * VNC to `5911` (display port `11`) with the standard OpenQA password.
    * SSH to `localhost` port `1123` with the username **`openqa`** and the standard OpenQA password.

**Note:** The "standard OpenQA password" can be obtained by talking to PatrickLightbody or any other developer who already has access.

# Build Server

Anyone currently can access build artifacts or see the state of the world inside Bamboo. There is another category of user who can configure builds and start manual ones. If you think you need access to do either of these things, let us know.

# Agents

All build VMs run a small Java program called the <dfn>build agent</dfn>, which takes instructions from the main server and send back build results and artifacts. These need to be started by hand when the machine restarts. Each of the machines has an `agent.sh` or `agent.bat` on them; typically in `C:\bamboo` or `/home/admin/bamboo-agent` or something very similar.

There are actually two agent jars that can be used. The most common one is the `installer` and has a restart-if-not-responding wrapper included. If that one is causing issues, there is one available on the build server agent download page that does not have it.