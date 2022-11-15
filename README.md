# htpc-setup  

## Some initial notes

All content contained herein is for educational purposes only.  Many of the Ansible Roles were written by other people, and the top of the Playbook makes note when applicable. If I missed anyone, please let me know so I can update it.

This is intended to be used on a from-scratch Debian install.  It probably works on other releases, but has only been tested in Debian Buster.

## Summary

Setting up a media server can be a bit of a bear with all the configuration involved.  This playbook aims to automate as much of that setup as possible, so you end up with several containerized instances of the applications you need to run your server.  Dependencies are installed initially, then the few things that must run outside containers are installed (MergerFS, Docker, SnapRaid, Discord and Cockpit).  A Docker-Compose file tells Docker what containers to pull and run from there.

Bare metal programs installed are:

* MergerFS - a file system that allows many physical drives to be pooled and used as one logical drive

* Docker - a system for running containerized programs

* SnapRaid - a parity file-check system to allow backup when one or more drives fail and monitoring otherwise

* Discord - a popular chat program, for which bots can be configured to allow users to make requests

* Cockpit - a monitoring program for the server

Containers do the following and use the noted ports (when applicable):  
<pre> 
localhost:port  

Application     Container Port      Default Port     What do I do?
--------------------------------------------------------------------

Cockpit           9090               9090            Monitoring application for system*
Gluetun           -----              -----           Allows use of VPN (Mullvad by default)
NZBGet            27020              6789            Usenet downloader
Sonarr            27021              8989            Television
Radarr            27022              7878            Movies
Prowlarr          27023              9696            Coordinate the *arrs and Deluge
Portainer         27024              9000            UI to keep an eye on containers
Deluge            27025              8112            Torrent client
Doplarr           -----              ----            Discord Bot to send requests to Overseerr
Overseerr         27026              5055            Allows requests and browsing for media 
Tautulli          27029              8181            Statistics for Plex
Heimdall          27030              80              Allows management of all webapps 
Unmanic           27050              8888            Converts media to set standard (optional)
Plex              32400              32400           Media server


*Not a container, running on actual hardware.

# Setup and configuration
  
</pre>

## Downloading

From the terminal, ensure you're in the directory where you wish for the repo to be saved, then type ```git clone https://github.com/scottglazer/public-htpc```

cd into the newly cloned directory.

## Initial preparation and setup

First, edit files from the repo:

### The hosts file:

This should work as-is with "localhost", but in case it doesn't, you can put your IP under the group designation, IE:

```
[localhost]
123.123.1.123
```

### The vars file

Edit the vars.yaml file to add:

#### Your SSH Private Key File 

This will look something like:

/home/user/.ssh/id_rsa

If you haven't generated one, go to the terminal and:

```

1.  Type "ssh-keygen".   You will be asked where to store it and under what name (by default /home/user/.ssh/id_rsa).  Press enter to save it.  

2.  Enter a passphrase (this is optional, but you should do it).

3.  Type "ssh-copy-id user@host" where "user" and "host" are the user you wish to run the playbook and the localhost respectively.  You can see these in the terminal by default, assuming you're logged in as that user.

4.  You will probably get a warning about the authenticity of the host not being able to be established. This is fine, you can type "yes" here.

5.  Enter your password.

That should be it.

```


#### Your SSH and sudo passwords

There is almost certainly a better way to do this than storing it in the vars file as plaintext (Ansible Vault, most likely), but this is what we're doing here.  When you've run the playbook, you can go back in and delete all this out if you're concerned; just be sure not to send a copy of the repo with any actual passwords in it to anyone.


#### An email address and password.  

Note that if you are using a gmail account, you will need to use an "app password" and not your normal password - you can set one up under the Security settings in your Google account.  Otherwise SnapRAID Runner won't be able to email out the results of the parity checks.

*NOTE: Any file beginning with a period is invisible by default in Linux.  To view the files in Debian, access the folder via the file manager and hit CTRL + H to toggle hidden files.  Other Linux distros may differ, but should have similar options.*


#### Information to configure MergerFS.  

MergerFS is used to pool multiple drives - that is, you can have multiple physical drives and have them appear as one giant drive consisting of the sum of the parts.  MergerFS handles all the logical elements of this, so if a file is too big for one component drive, whatever doesn't fit will be saved to another.  You will only see it as one file in the OS.  More drives can be added at a later time and added to the pool without much effort.

You will still need the vars file for the next step, but FSTAB must be configured before you make those edits, so leave it open in the editor for now.


##### FSTAB

This requires a bit of setup outside the vars file - you need to edit your fstab entry to mount each drive first.

The vars file itself gives brief instructions in the comments, which I'll repeat here:

<pre>
 MERGERFS

Edit these to match your mounts.
You will want to edit /etc/fstab to add the drives.
"sudo blkid" to list all drive UUID's.
Add these to fstab using format:

 <file system>                                            <mount point>         <type>      <options>         <dump>  <pass>
UUID=aad41a23-be97-4cc8-a54c-51238af932b1                  /mnt/disk1            ext4        defaults          0       0
</pre>

##### MergerFS vars

The actual vars need to be configured in the following format:

```
Configuration setting:                  Description:
----------------------------------------------------------------------------------
mergerfs_mounts:
  - path: /mnt/storage                  The path and name used by the pool itself.
    branches:                           The components of the pool by mount point.
       - /mnt/disk1                     Component 1
       - /mnt/disk2                     Component 2
       - /mnt/disk3                     Component 3
    options: direct_io,defaults...      Related options for the pool.
```

In this case, you'll be able to look at the drive "storage" and see everything held within disks 1 through 3 as though they were on one drive.

Once the playbook has been run, you can add more drives by just adding them to the playbook and re-running, or by following the instructions in the comments of the vars file and editing /etc/fstab manually (since you'll need to add the actual drives by UUID manually anyhow).


#### Vars for SnapRAID and SnapRAID-Runner

SnapRAID 

The parity entry indicates the drive(s) you wish to use for parity. More than one can be used in case of multiple drive failure, but they must be the largest drives in the array.

The snapraid.content files are listings and details of the content of the disks. 
The data entries specify the disks to check for data.

Your parity and content drives need to be assigned.  The format will be looking to the mount point, with the role assignment preceding it.  By way of example:

snapraid_data_disks:
  - path: /mnt/disk1
    content: true

and:

snapraid_parity_disks:
  - path: /mnt/parity1
    content: true
 


### The .env file

Update the .env file with your personal information to be pulled into the docker-compose when it runs. At this point, you may only have enough to fill out the PUID, PGID, EMAIL and (if you're using it) MULLVAD_ID.  Save the file now. You'll return to it later on during app setup if you intend to set up Doplarr.


VPN setup is handled via Mullvad. If you're not using Mullvad, I suggest you determine a VPN to use via docker container, but setup for Mullvad in particular only requires that you edit the MULLVAD_ID variable in the .env file.  If you don't intend to use Mullvad, the next section (on the docker-compose file) explains how to disable that container.

Docker containers should be run with a user that can sudo.  Assuming that's your current user, run a whoami command (to show the username), 
then id [username], where [username] is what you found from the whoami.  Edit the .env file so PUID and PGID are equal to what you saw under uid
and gid respectively.


## The docker-compose file

Double check the docker-compose file to edit anything that might differ on your system or that you might wish to otherwise change.  Pay particular attention to the volumes.  For any given volume, the format
is:

actual/volume/location/on/pc:volume/seen/in/docker

So, for example, one default volume mapping for NZBGet is:

- /opt/appdata/nzbget/config:/config

/opt/appdata/nzbget/config is the folder where data is actually located on your PC.
/config is how the container refers to that folder.

It's important to bear this in mind with setup, since configuration (the next step) will refer to the 
*container* file paths and not the paths on the PC itself.

If you do not wish to use a VPN, comment out the entire Gluetun entry and paste the port mapping (other than the first four, so starting at portainer) into each individual service so it knows what ports to use.

## First Run and Configuration

The Docker-Compose will run as soon as the playbook is done running, and any required directories will be created.  Briefly, to control the docker-compose stack from the directory where docker-compose.yaml is located, you can type the commands:

  docker-compose pull:  updates all docker files
  docker-compose up -d: starts the stack in the background
  docker-compose stop:  stops the stack
  docker-compose down:  stops the stack and removes the containers
  docker-compose rm -f: removes all containers

If you run into issues later on, it can be helpful to know how to get rid of non-running "ghost" containers:

  docker rm $(docker ps -a -q)    Will check for containers.
  docker rmi $(docker images -aq) Will remove any that should not be present.

Upon first run, you will need to configure NZBGet and the -arrs.  This is mostly done via browser, connecting to each application on its port.  

I would recommend doing so in this order:


### Set up NZBGet.  
Connect to http://localhost:27020/  Under the settings tab, define:

  ```
       A: Paths.  If you wish to use something other than the default paths, 
        you will need to edit the docker-compose file beforehand so the 
        container can access that path. 
       
       B: News-Servers. You should enter whatever providers you have here.  
        Make sure your connections are less than or equal to the actual max 
        connections your provider gives (I would do max - 2 or so, to be sure 
        you don't end up being warned or losing access to the provider 
        temporarily).
       
       C: Security: define a control username and password to prevent 
        unauthorized access.
  ```

### Set up Sonarr and Radarr.  
Connect to http://localhost:27021/ (Sonarr) followed by http://localhost:27022/ (Radarr).  Under the settings tab:
       
  ```
	   A:  Choose the General category and take note of your API key for
	    when you set up Prowlarr (alternatively, just leave the tab open 
	    for when you configure Prowlarr later so you can just click the copy
	    button and paste it in).
	   
	   B:  Ensure the Media Management, Profiles and Quality tabs are set to 
	    what you wish to use.  Unless you have a ton of space, remove remux 
	    from any of the quality profiles.  Set up the root folder path (choose 
	    /tv for television and /movies for movies).

  ```

### Set up Prowlarr. 

                ** A Note **

        If you chose not to use a VPN, you will need to configure the mapped 
        ports rather than the internal ones- so port 27020 rather than 6789 
        and the host should be the IP address of the PC itself (though 
        localhost should also work).  Bear this in mind with the following 
        (which are written assuming you're using the VPN):

Connect to http://localhost:27023/  Define:
       
  ```


       A: Download Clients under the settings tab.  Choose NZBGet, use the 
        host http://localhost on port 6789.  If you configured NZBGet 
        to use a login/password, define those here as well.  Click "Test", 
        and assuming the test passes, save from there.

       B: Applications under the settings tab.  Configure Radarr and Sonarr 
        individually, using http://localhost:8989 for Sonarr and 
        http://localhost:7878 for Radarr, replacing with the IP of the PC 
        itself.  Test and save.  Click "Sync App Indexers" once setup of both
        apps is done.
       
       C: Indexers under the indexers tab.  You'll need the indexer URL and
        API, the latter of which is usually found on your user profile page 
        (this can vary by indexer).  Specifically ensure the checkbox 
        labelled "Redirect incoming download requesat for indexer and pass 
        the grab directly instead of proxying the request via Prowlarr" 
        is checked.  If it is not, many indexers may block the connections 
        after a period of time.


   ```
  
### Verify VPN is working from Portainer.  
Connect to http://localhost:27024.  

  If this is the first time you've used Portainer, it will ask you to define a password.  Don't lose this password - resetting it requires you search out the Portainer database and delete it (with the containers halted).  Once you're signed in:
```
    Click on the local environment (should be the only one, has an icon that looks like a whale with a bunch of boxes on top of it).
    Click "Containers"
    Click on the >_ symbol in the portainer row.
    Click Connect - this will ssh you into the container.
    Type:
      curl ifconfig.io
    You should see an IP address returned that is not on your local network. This is the address used by the VPN.  
    Assuming all's well, you can type "exit" to get out of the session.
```

### Configure Deluge.  
Connect to http://localhost:27025.  The default password is simply "deluge".  You should change it (there's a popup asking to do this when it's first started as well).  

```
  In the Preferences, click the Network tab and enter port 6881 under "Incoming Port", ensuring "Use Random Port" is left unchecked.  Ensure "Outgoing Port is set to 6882 in a similar manner.  Under the   
Downloads tab, configure "Download to:" to /downloads.  Feel free to change other settings as necessary.
```  

  ***Configuring Overseerr and Doplarr is only necessary if you wish to have users request media via Discord. Overseerr can be used standalone as well, if you just like a nice interface to request things from***


### Configure Overseerr.  
Connect to http://localhost:27026.  Log into your plex account, select the server (if you have more than one you may want to manually do this), click "save changes", "then continue".

```
  Configure the Radarr and Sonarr servers. Copy your API key from Radarr (under Settings -> General), enter localhost as your  address and leave the default ports alone (this is communicating within the container).  Give the server a name, test it, then fill out the quality dropdowns (make sure "Enable Scan" and "Enable Automatic Search" are both checked)and save. These settings can be edited later under Settings -> Services.

  Under "Settings," click the Plex tab and click to refresh the list of servers if the list isn't already available.  Once that's been done, sync the libraries and (if you have existing media) do a manual library scan.
```

### Configure Doplarr. 
Modified instructions from [here](https://kiranshila.github.io/Doplarr/#/configuration):


```
There are four .env variables to fill out, as well as some configuration throughout.  Variables are boldfaced.

In Discord:

Create a new [Application](https://discord.com/developers/applications) in Discord
Go to the Bot tab and add a new bot.
Copy the token, enter it as the *DISCORD_TOKEN* in the .env file.
Go to OAuth2 and under "OAuth2 URL Generator", enable applications.commands and bot.
Copy the resulting URL and use as the invite link to your server.

    In the server for which you will use the bot, you have the option to restrict the commands to a certain role. For this, you need to create a new role for your users (or use an existing one). Then, grab that role id.

To do this:

    Enable Developer Mode (User Settings -> Advanced -> Developer Mode)
    Under your server settings, go to Roles, find the role and "Copy ID"

    Paste this value under *ROLE_ID* in the .env file.

    Every user that you wish to have access to the slash commands needs to be assigned this role (even the server owner/admins).  Make sure you assign yourself this role as well!

Run the docker-compose (if it's not already running, in which case stop and restart the stack) and copy the Overseerr API key (Settings -> General ->API Key, copy).  Paste this in for the *OVERSEERR_API* value.

While still in Overseerr, create a new user (click "Users", then "Create Local User").  Copy the Discord User ID into the field of the same name (it's found in Discord by right clicking the Doplarr bot and then "Copy User ID", which is the last option in the list).  Save, then edit the user again and make note of the User ID (which is right under the name and icon for the user group).  Paste this into the .env file under *OVERSEER_ID*.

```

### Configure Tautulli.  
Connect to http://localhost:27029.  Follow the Setup Wizard.

### Configure Heimdall.  
Connect to http://localhost:27030.  Add and pin applications as desired, so you don't have a dozen bookmarks to everything.

### Use Unmanic (Optional)
If you wish to use Unmanic, you'll need to uncomment it from the docker-compose before running the docker-compose pull command.

You can use Unmanic to modify your media to match one unified format (and to monitor anything coming in, if you wish).  Connect to it by going to http://localhost:27050

```
From the Home screen, click the three lines in the upper left-hand corner.  
Select Settings.
In the Library tab, the library itself will be pointing to /library.  You can add two more libraries for TV and Movies (using /tv and /movies respectively - the true paths are in the docker-compose).
Click on the Plugins tab, choose plugins that suit your purpose.

```

# Post-setup and Usage:

Once the Playbook has been run and everything has installed, there is no longer any need to run it.  You'll be running against the docker-compose file from here on out.

### Docker-Compose

To reiterate what was noted earlier:  to control the docker-compose stack from the directory where docker-compose.yaml is located, you can type the commands:

  docker-compose pull:  updates all docker files
  docker-compose up -d: starts the stack in the background
  docker-compose stop:  stops the stack
  docker-compose down:  stops the stack and removes the containers
  docker-compose rm -f: removes all containers

### Snapraid

Once the playbook has been run, initialize SnapRAID by running the "snapraid sync" command (this may require sudo permissions).  This may take several hours, depending on the size of your pool and what's in it, but subsequent runs will take much less time.

Other manual snapraid commands worth knowing are:

snapraid scrub - verifies data in array.
snapraid status - gives details on any I/O errors found.
snapraid -e fix - will attempt to fix any errors.
snapraid -p bad scrub - will remove errors from status report, assuming they are fixed.

More details and explanation of recovery options can be found at https://www.snapraid.it/manual

### Snapraid-Runner
This can be run manually by going to /opt/snapraid-runner and typing:
```
sudo python3 snapraid-runner.py
```
...which is worth doing to verify the script is working.

The playbook should've added a cron job to ensure Snapraid runs as specified in the vars file.  To check this, you can just check your cron jobs with the "crontab -e" command.  








