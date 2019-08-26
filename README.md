# Samba share docker service

I made a docker swarm linux service to expose a rexray/s3fs volume `rhlab-samba-share` to my swarm. 

This worked for me in Server 2019 core with containers and Amazon linux 2. 

On each linux swarm instance run:

```
sudo yum update
sudo yum install docker
sudo usermod -a -G docker ec2-user
sudo systemctl start docker
sudo docker plugin install rexray/s3fs S3FS_ACCESSKEY=<yours> S3FS_SECRETKEY=<yours>

```
Then create your swarm using [these instructions](https://caylent.com/high-availability-docker-swarm-aws/).

On a swarm master from [these instructions](https://github.com/dperson/samba) run:

```

docker service create --name samba -p 139:139 -p 445:445  \
	--mount type=volume,source=/g,destination=/myvol,volume-label="color=red",volume-label="shape=round" \
       dperson/samba -u "example1;badpass" \
-s "data;/myvol;no;no;no;example1" -p    

```

Then on my linux swarm instances I followed [these instructions](http://timlehr.com/auto-mount-samba-cifs-shares-via-fstab-on-linux/) and added this to my /etc/fstab:

```

//localhost/data        /g      cifs    uid=0,credentials=/home/ec2-user/.smb,iocharset=utf8,vers=3.0,noperm 0 0

```

On my windows swarm instances I plan to join the swarm as a worker and create this `C:\data\smbshare.ps1`:

```

$secpasswd = ConvertTo-SecureString 'badpass' -AsPlainText -Force;
$creds = New-Object System.Management.Automation.PSCredential ("example1", $secpasswd);
New-SmbGlobalMapping -RemotePath '\\localhost\data' -Credential $creds -LocalPath G:;


```

Then make a cmdlet:

```

$Action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument "-file C:\data\smbshare.ps1" -WorkingDirectory "C:\data";
$Trigger = New-ScheduledTaskTrigger -AtStartup;
$Settings = New-ScheduledTaskSettingsSet -DontStopOnIdleEnd -RestartInterval (New-TimeSpan -Minutes 1) -RestartCount 10 -StartWhenAvailable;
$Settings.ExecutionTimeLimit = "PT0S";

```

Based on [this stack overflow question](https://stackoverflow.com/questions/50415447/smb-share-mappings-created-with-new-smbglobalmapping-for-docker-containers-not-r).
