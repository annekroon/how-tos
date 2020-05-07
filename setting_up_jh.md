## install juypterhub on Ubuntu-18.00

This tutorial relies on: https://bbengfort.github.io/tutorials/2019/10/09/launch-jupyterhub-server.html

### Step 0: install packages

```
sudo apt install emacs
```

### Step 1: disable firewalls

disable firewalls for the moment -- you want to reable again last
```
sudo ufw disable
```
### Step 2: Install Anaconda

>I chose to use Anaconda to facilitate data science workloads for this installation. Anaconda has its pros and cons on a system level
>install, but one of the implications was that users could not install their own packages. Additionally, I had to jump through some >
>hoops to get the server running with systemd.
>After this experience, vanilla Python might be a better choice, to be frank.
>First create a system user for Anaconda and add the ubuntu user to the group:

```
sudo useradd -r anaconda
sudo usermod -a -G anaconda ubuntu
```

Next install Anaconda as follows:
1. Select distribution from Anaconda Distributions
2. Copy 64-bit (x86) installer URL
3. Download, verify integrity, and execute the script

```
curl -O https://repo.anaconda.com/archive/Anaconda3-2019.07-Linux-x86_64.sh
sha256sum Anaconda3-2019.07-Linux-x86_64.sh
#69581cf739365ec7fb95608eef694ba959d7d33b36eb961953f2b82cb25bdf5a  Anaconda3-2019.07-Linux-x86_64.sh
sudo bash Anaconda3-2019.07-Linux-x86_64.sh
```
4. Accept the user agreement
5. Install to `/opt/anaconda`
6. 'yes' to init conda
7. Update permissions of `/opt/anaconda`

```
source /opt/anaconda/bin/activate`
conda init`

sudo chown -R anaconda:anaconda /opt/anaconda
sudo chmod -R 775 /opt/anaconda

```
These permissions should give the ubuntu user the ability to install packages to anaconda, but not other users. Other users should be able to execute anaconda commands but not modify the anaconda install. If they want to install their own packages, they’ll have to create a virtual environment in their home directory.

Optional step: ensure Anaconda is available for new users
This is an optional step, but because Anaconda relies heavily on the shell for configuration,
I ensured that any new users would have access to Anaconda by appending the following to `/etc/skel/.bashrc`:

```
sudo emacs /etc/skel/.bashrc
sudo emacs /root/.bashrc
```

```bash
# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/opt/anaconda/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/opt/anaconda/etc/profile.d/conda.sh" ]; then
        . "/opt/anaconda/etc/profile.d/conda.sh"
    else
        export PATH="/opt/anaconda/bin:$PATH"
    fi
fi
unset __conda_setup

# <<< conda initialize <<<
```

Note that I also added this to `/root/.bashrc` so that when I executed commands with `sudo su`, Anaconda was available to the root user.


### Step 3: Install JupyterHub
Install the required packages using conda and pip:

```
sudo chown -R ubuntu /opt/anaconda
```

```

(base) $ conda install -c conda-forge jupyterhub
(base) $ conda install notebook
(base) $ pip install jupyterhub-systemdspawner
```
```
sudo useradd -r jupyterhub
sudo usermod -a -G jupyterhub ubuntu
sudo mkdir /srv/jupyterhub
sudo chown -R jupyterhub:jupyterhub /srv/jupyterhub
```

### Step 4: letsencrypt

```
sudo apt install certbot
sudo certbot certonly
```

Appropriate number == 1 #  ('set up temporary web server')
Fill in your email address, agree with conditions, sign up for e-mail (no), register your {domain-name}

If successfull, you read something like:

```
Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/{domain-name}/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/{domain-name}/privkey.pem

```

Save this information somewhere.

### Step 5: Configure juypterhub

Create the jupyterhub configuration file and move it to the recommended system configuration location as follows:

```
jupyterhub --generate-config
sudo mkdir /etc/jupyterhub
sudo mv jupyterhub_config.py /etc/jupyterhub
```

Add the following info to `/etc/jupyterhub/jupyterhub_config.py`:
`sudo emacs /etc/jupyterhub/jupyterhub_config.py`

```
c.JupyterHub.ssl_cert = '/etc/letsencrypt/live/{domain}/fullchain.pem'
c.JupyterHub.ssl_key = '/etc/letsencrypt/live/{domain}/privkey.pem'

c.Authenticator.admin_users = set(['ubuntu', 'admin'])
```

Create a new user and password for the admin user:

```
sudo adduser admin
```

Enter the password and details for the admin user – this will give you admin access to the JupyterHub server and allow you to create new users online.

At this point, as the root user you should be able to run:

#### set 'rights right'

```
sudo chown ubuntu.ubuntu -R /srv/
sudo chown ubuntu.ubuntu -R /etc/letsencrypt/
```

```
jupyterhub -f /etc/jupyterhub/jupyterhub_config.py
```

You should be able to see the login screen and login as the admin user. You should also be able to create new users from the admin page.


### Step 6: Run JupyterHub as a systemd service

Because anaconda needs to be initialized and the environment modified to support it, the systemd service needs to be run from a bash script. Add the following to `/usr/local/bin/run_jupyterhub.sh` and make it executable.

`sudo emacs /usr/local/bin/run_jupyterhub.sh`


```bash
#!/bin/bash

# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/opt/anaconda/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/opt/anaconda/etc/profile.d/conda.sh" ]; then
        . "/opt/anaconda/etc/profile.d/conda.sh"
    else
        export PATH="/opt/anaconda/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<

# Run the JupyterHub service with the system configuration
jupyterhub -f /etc/jupyterhub/jupyterhub_config.py
```

Make executable:
`sudo chmod u+x /usr/local/bin/run_jupyterhub.sh`

Create a systemd service by writing the following into `/etc/systemd/system/jupyterhub.service`:
`sudo emacs /etc/systemd/system/jupyterhub.service`

```
[Unit]
Description=JupyterHub
Documentation=https://jupyterhub.readthedocs.io/en/stable/

[Service]
Type=simple
After=network.target
Restart=always
RestartSec=10

WorkingDirectory=/srv/jupyterhub
ExecStart=/usr/local/bin/run_jupyterhub.sh

StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=jupyterhub

[Install]
WantedBy=multi-user.target
```

You can now enable and start the server as follows:

```
$ sudo systemctl enable jupyterhub.service
$ sudo systemctl start jupyterhub.service
```

You should now be able to go to the server and access JupyterHub without running it specifically as the user. To diagnose issues, use journalctl as follows:
```
$ sudo journalctl -u jupyterhub.service
```

```
https://{domainname}:8000/login
ak.recruitmentbias-uva.surf-hosted.nl:8000/login
```


#### close ports again

sudo ufw allow 443
sudo ufw allow 8000
sudo ufw enable

####  Checking available storage

df -h
df -Ph . | tail -1 | awk '{print $4}'

create symoblic link:
`sudo ln -s /mnt/data`
