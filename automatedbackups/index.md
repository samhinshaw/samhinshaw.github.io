## Automated Backups with Globus and crontab!

### I. Setting up Globus

1. Go to [globus.org](www.globus.org) and login or sign up with your Westgrid ID (OAuth).  
2. Go to [globusid.org](www.globusid.org) and create an account with an email address @globusid.org.  
3. At [globusid.org](www.globusid.org), [Add a new SSH key](https://docs.globus.org/cli/).  
	+ I always like to use [GitHub's instructions](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/) for this step, as I always forget the commands!  Briefly, in your local terminal:  
	
```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
< enter /path/to/SSHkey >
< enter passphrase >
eval "$(ssh-agent -s)"
ssh-add < /path/to/SSHkey >
```

4. Now go back to globus.org's [account section](https://www.globus.org/app/account), and click "Add Linked Identity".  
5. In the box, type "Globus ID", and proceed through the OAuth authorization.  
6. Now test your ssh connection! In your local terminal:  

```
ssh <globusid.org username>@cli.globusonline.org
```

Now you should have access! You can type `help` to check out the command line options.  
	
> Note: You are now logged into the Globus System.  From here you can access various endpoints (e.g. WestGrid resources).  For now though, we don't need to do anything!  Let's download the globusconnect client so that our local machine can be turned into an globus endpoint.  

You can read more about the Globus CLI Interface [here](https://docs.globus.org/cli/using-the-cli/) and [here](https://docs.globus.org/cli/cli-beyond-basics/). 

### II. Globus Connect  

1. Visit the [Globus Connect Personal](https://www.globus.org/globus-connect-personal) website. For instructions specific to Linux, go [here](https://docs.globus.org/how-to/globus-connect-personal-linux/).  You may have to install Tcl/Tk dependencies, tcllib.
	- Follow the directions on this website to run your initial globusconnect setup.  
	- If you still have problems with tcllib dependencies, you can check out [the instructions here](https://docs.globus.org/how-to/globus-connect-personal-cli/) for setting up globus personal connect exclusively via command line. Thanks to Amy for the tip! @minisciencegirl 
2. Now close your globusconnect GUI & move your unzipped/untarred directory to wherever you'd like the installation to live.  I moved mine to ~/bin/globus/, and then added that to my system PATH via ~/.profile:  
```
if [ -d "$HOME/bin/globus" ] ; then
    PATH="$HOME/bin/globus:$PATH"
fi
```  

3. If you [continue down the page](https://docs.globus.org/how-to/globus-connect-personal-linux/), you can see how to run globusconnect from the command line!  But first let's make sure `globusconnect` is executable:  
```
chmod +x globusconnect
```  

Finally, let's make sure `globusconnect` is now a command available in our system PATH. 
```
echo $PATH
globusconnect -start &
globusconnect -status
```

Great! Now on to our script.  

### III. Backup Script

Put this script wherever you might keep scripts you're planning to run often.  I created a directory in my home folder, ~/scripts, and stored this as globus_sync.sh.  I've copied the file into my root directory, but I've pasted it here as well.  

```
#!/bin/bash

if [[ "$(globusconnect -status | head -n 1)" == "Globus Online:   connected" ]]; then
    echo "Globus Connect is Already Online!"
else
    globusconnect -start &
    echo "Globus Connect Started!"
    sleep 1
    globusconnect -status 
fi

ssh samhinshaw@cli.globusonline.org endpoint-activate computecanada#bugaboo

if [[ "$(ssh samhinshaw@cli.globusonline.org status)" ]]; then
    echo "backup already running, exiting..."
    exit
else
    echo "u_7ayeuixr7ii6lpnawmwbjkxxwm#SamWorkstation/~/Dropbox/hancocklab/ computecanada#bugaboo/~/hancocklab/ -r" | ssh samhinshaw@cli.globusonline.org transfer --encrypt --preserve-mtime --verify-checksum -s 3 
fi

ssh samhinshaw@cli.globusonline.org status

while [[ "$(ssh samhinshaw@cli.globusonline.org status)" ]];
do
    echo "transferring..."
    sleep 1m
done;
    globusconnect -stop 

exit
```

Important things to change:  

- Your GlobusID username  
- Your personal endpoint name.  Mine is `u_7ayeuixr7ii6lpnawmwbjkxxwm#SamWorkstation`.  Find yours at [globus.org > Manage Data > Endpoints](https://www.globus.org/app/endpoints).  
	- Click your computer's name and find the "Legacy Name" at the very bottom.  
- Your personal endpoint directory.  I've started by just backing up my hancocklab folder at "~/Dropbox/hancocklab/"  
	+ Note: if you run into permissions errors, check out [the instructions here](https://docs.globus.org/faq/globus-connect-endpoints/#how_do_i_configure_accessible_directories_on_globus_connect_personal_for_linux) for granting globus read or write permissions to specific directories.  This is particularly relevant if you're backing up files stored in /mnt/. Thanks again to Amy for this tip! @minisciencegirl  
- The endpoint you wish to transfer your files to. I'm using bugaboo, a server at SFU.  
- The endpoint directory you wish to transfer to.  I'm using "~/hancocklab/" on bugaboo.  
- Your transfer flags:  
	+ `-r` recursive, will transfer all files and subdirectories recursively.  
	+ `--encrypt` will encrypt the **transfer**, not the files.  
	+ `--preserve-mtime` will preserve the "modified time" metadata of your files  
	+ `--verify-checksum` will hash your files to make sure the files transferred successfully  
	+ `-s 3` this is the "depth" of your transfer. not specifying a flag will cause all files to transfer, overwriting previous content. 3 compares file hashes to determine if a transfer is necessary.  Various other depths are available, such as file size-based comparisons.  
	+ `--delete` will delete any other files in your destination endpoint not part of this transfer  
Other options are available, and can be examined by SSHing into globus:
```
ssh samhinshaw@cli.globusonline.org
transfer -h
```  

Potential additions:  

- creating a "daily backup" label with the `-l` flag (I wasn't able to get this working)  
- decide on a method for deleting older files no longer needed on the target endpoint as to not clutter the system with deleted files.  
	
After you've saved this, make sure it's executable, as we did with `globusconnect`:
```
chmod +x globus_sync.sh
```

### IV. Automation  

1. Install keychain.  This is necessary to allow crontab to access your ssh keys when executing the script.  I followed the instructions in [this stackoverflow thread](http://stackoverflow.com/questions/869589/why-ssh-fails-from-crontab-but-succedes-when-executed-from-a-command-line).  
```
sudo apt-get install keychain
keychain --nogui < SSH Key Name >
```  



2. Now, we'll add a command to your personal login profile, ~/.profile, to start keychain and add your SSH Key for globus at login.  
- Install askpass so that we'll get a prompt at login to input your passphrase protecting your private SHH key.  

```
sudo apt-get install ssh-askpass-gnome ssh-askpass
```  

- At the bottom of ~/.profile, add:

```
export SSH_ASKPASS=/usr/bin/ssh-askpass
eval `keychain --eval globusid`
```  

3. . Run `crontab -e` to access your local crontab settings. Under the line
```
# m h  dom mon dow   command
```  
- I have the following code:
```
SHELL=/bin/bash
PATH=/home/shinshaw/bin/globus:/home/shinshaw/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/bin/globus

00 04 * * * . ~/.keychain/$HOSTNAME-sh &>> /home/shinshaw/scripts/backup.log; /home/shinshaw/scripts/globus_sync.sh &>> /home/shinshaw/scripts/backup.log

```  
- For some reason crontab wants me to FULLY specify my path, so I've just copy + pasted my output from:
```
echo $PATH
```  
- Also, you need to specify `/bin/bash`, as crontab's default behavior is to use `sh` rather than `bash`.  

- I have my script set up to run daily at 4am, but you can set it to run as often as you want!  If a backup is already going, then the script should abort.  

> Parting Note: Whenever you perform a successful backup, you'll get an email from Globus (to your WestGrid-registered email address) giving you the details!  Make sure you have your notifications muted so you don't get emails waking you up at 4am :)

