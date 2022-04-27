# htpc-setup  

## Some initial notes

All content contained herein is for educational purposes only.  Many of the Ansible Roles were written by other people, and the top of the Playbook makes note when applicable. If I missed anyone, please let me know so I can update it.

This is intended to be used on a from-scratch Debian install.  It probably works on other releases, but has only been tested in Debian Buster.

## Summary

Installs dependencies and required files for MergerFS, Docker, Snapraid, Discord and Cockpit to run on bare metal, then uses docker to run everything else.

Containers will use the following for connections:  
<pre> 
localhost:port  

Application     Container Port      Default Port     What do I do?
--------------------------------------------------------------------

Cockpit           9090               9090            Monitoring application for system*
NZBGet            27020              6789            Usenet downloader
Sonarr            27021              8989            Television
Radarr            27022              7878            Movies
Prowlarr          27023              9696            Coordinate the *arrs
Portainer         27024              9000            UI to keep an eye on containers
Deluge            27025              8112            Torrent client
Overseerr         27026              5055            For Requestrr to send to 
Tautulli          27029              8181            Statistics for Plex
Heimdall          27030              80              Allows management of all webapps 
Unmanic           27050              8888            Converts media to set standard
Plex              32400              32400           Media server

*Not a container, running on actual hardware.
  
</pre>

## Initial preparation and setup

1. Edit files from the repo:

Add your IP to the hosts file under [primary].

Edit the vars.yaml file to add an email address and password.  You'll be editing it a bit more in step 2, so you may as well leave it open.

*NOTE: Any file beginning with a period is invisible by default in Linux.  To view the files in Debian, access the folder via the file manager and hit CTRL + H to toggle hidden files.  Other Linux distros may differ, but should have similar options.*

Update the .env file with your personal information to be pulled into the docker-compose when it runs. At this point, you may only have enough to fill out the PUID, PGID, EMAIL and (if you're using it) MULLVAD_ID.  Save the file now. You'll return to it later on during app setup if you intend to set up Doplarr.

2.  You will need to configure MergerFS.  The vars file gives brief instructions in the comments, which I'll repeat here:

<pre>

 MERGERFS

Edit these to match your mounts.
You will want to edit /etc/fstab to add the drives.
"sudo blkid" to list all drive UUID's.
Add these to fstab using format:

 <file system>                                            <mount point>         <type>      <options>         <dump>  <pass>
UUID=aad41a23-be97-4cc8-a54c-51238af932b1                  /mnt/disk1            ext4        defaults          0       0

...for each drive (iterating the mount point: disk2, disk3, etc), with the pooled drive defined as:

     mergerfs pool 
 <file system>  <mount point >  <type>         <options>                                                        <dump>  <pass> 
 /mnt/disk*     /mnt/storage    fuse.mergerfs  direct_io,defaults,allow_other,minfreespace=50G,fsname=mergerfs  0       0

</pre>

3.  Docker containers should be run with a user that can sudo.  Assuming that's your current user, run a whoami command (to show the username), 
then id [username], where [username] is what you found from the whoami.  Edit the .env file so PUID and PGID are equal to what you saw under uid
and gid respectively.

4.  Double check the docker-compose file to edit anything that might differ on your system or that you might wish to otherwise change.  Pay particular attention to the volumes.  For any given volume, the format
is:

actual/volume/location/on/pc:volume/seen/in/docker

So, for example, one default volume mapping for NZBGet is:

- /opt/appdata/nzbget/config:/config

/opt/appdata/nzbget/config is the folder where data is actually located on your PC.
/config is how the container refers to that folder.

It's important to bear this in mind with setup, since configuration (the next step) will refer to the 
*container* file paths and not the paths on the PC itself.

VPN setup is handled via Mullvad. If you're not using Mullvad, I suggest you determine a VPN to use via docker container, but setup for Mullvad in particular only requires that you edit the MULLVAD_ID variable in the .env file.

If you do not wish to use a VPN, comment out the entire Gluetun entry and paste the port mapping (other than the first four, so starting at portainer) into each individual service so it knows what ports to use.

5.  The Docker-Compose will run as soon as the playbook is done running, and any required directories will be created.  Briefly, to control the docker-compose stack from the directory where docker-compose.yaml is located:

  docker-compose pull:  updates all docker files
  docker-compose up -d: starts the stack in the background
  docker-compose stop:  stops the stack
  docker-compose down:  stops the stack and removes the containers
  docker-compose rm -f: removes all containers

If you run into issues later on, it can be helpful to know how to get rid of non-running "ghost" containers:

  docker rm $(docker ps -a -q)    Will check for containers.
  docker rmi $(docker images -aq) Will remove any that should not be present.

## CONFIGURATION AND SETUP OF APPS

Upon first run, you will need to configure NZBGet and the -arrs.  I would recommend this order:



### Set up NZBGet.  
Connect to http://localhost:27020/  Under the settings tab, define:

  ```
       A: Paths.  Bear in mind the warning in step 4.  If you wish to use
        something other than the default paths, you will need to edit the 
        docker-compose file beforehand so the container can access that path. 
       
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
Connect to http://localhost:27023/  Define:
       
  ```
                ** A Note **

        If you chose not to use a VPN, you will need to configure the mapped 
        ports rather than the internal ones- so port 27020 rather than 6789 
        and the host should be the IP address of the PC itself (though 
        localhost should also work).  Bear this in mind with the following 
        (which are written assuming you're using the VPN):

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

  In the Preferences, click the Network tab and enter port 6881 under "Incoming Port", ensuring "Use Random Port" is left unchecked.  Ensure "Outgoing Port is set to 6882 in a similar manner.  Under the   
Downloads tab, configure "Download to:" to /downloads.  Feel free to change other settings as necessary.
  

  ***Configuring Overseerr and Doplarr is only necessary if you wish to have users request media via Discord. Overseerr can be used standalone as well, if you just like a nice interface to request things from***


### Configure Overseerr.  
Connect to http://localhost:27026.  Log into your plex account, select the server (if you have more than one you may want to manually do this), click "save changes", "then continue".

  Configure the Radarr and Sonarr servers. Copy your API key from Radarr (under Settings -> General), enter localhost as your  address and leave the default ports alone (this is communicating within the container).  Give the server a name, test it, then fill out the quality dropdowns (make sure "Enable Scan" and "Enable Automatic Search" are both checked)and save. These settings can be edited later under Settings -> Services.

  Under "Settings," click the Plex tab and click to refresh the list of servers if the list isn't already available.  Once that's been done, sync the libraries and (if you have existing media) do a manual library scan.


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
You can use Unmanic to modify your media to match one unified format (and to monitor anything coming in, if you wish).  Connect to it by going to http://localhost:27050









