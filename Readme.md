# BoxBackup on Docker

## Motivation

I find setting up BoxBackup an involved step with many confusing steps, particularly 
with certificates. The final result is not overly mobile. And you have to do it all over 
again when your OS goes through an upgrade. Can we use Docker to make the 
whole thing simpler? Yes: Using Docker we can isolate the BoxBackup server from the 
underlying Operating System. In fact since it is stateless we can swap out the server
so long as the new server can see the disk. This might for instance allow us to 
use a slow Raspberry Pi to continuously back up data to an external disk and
in an emergency connect the disk to a much faster computer and do the restore from there.

## Server Concept

The idea is that the BoxBackup server is stateless. It requires the client connections
to come in on port 2201 and needs to see a disk directory for its state. Docker allows
us to specify these two requirements in the `docker run` statement.

In this example we are forwarding the host's port 2201 to the daemon process and
forwarding the directory `/usb-disk/boxbackup-data` to the container's local
/boxbackup-data directory. This opens up a lot of configuration options as the
administrator can now use a different port on the host for the incoming connections
or can chose from a variety of hardware and OS specific options for the persistent 
storage.

```bash
docker run -d --restart=always \
	--hostname boxbackup \
	-v /usb-disk/boxbackup-data/:/boxbackup-data/ \
	-p 2201:2201 \
	--rm=false \
	richardeigenmann/boxbackupdaemon-arm
```

## Setting up the server

Set up the disk directory that you want to mount into the container.
It needs to contain
* `accounts.txt` the text file with the accounts
* `server-cert.pem` the server's public certificate 
* `server-key.pem` the server's private certificate 
* `serverCA.pem` the server's public SSL certificate 

Using the `docker run` command with the `--restart=always` option will tell
Docker to always start our daemon container when Docker comes up. You need
to make sure the host starts Docker.

## Creating the certs

```bash
bbstored-certs ca init
```

This creates these files:

```
./ca/roots
./ca/roots/clientCA.pem
./ca/roots/clientCA.srl
./ca/roots/serverCA.pem
./ca/roots/serverCA.srl
./ca/keys
./ca/keys/clientRootKey.pem
./ca/keys/clientRootCSR.pem
./ca/keys/serverRootKey.pem
./ca/keys/serverRootCSR.pem
```

Then

```bash
mkdir workdir
# create a raidfile.conf
# create a user boxbackup
sudo bbstored-config workdir server.hostname boxbackup
```

You get:


```bash
===================================================================

bbstored basic configuration complete.

What you need to do now...

1) Sign gaga/bbstored/host.server.net-csr.pem
   using the bbstored-certs utility.

2) Install the server certificate and root CA certificate as
      gaga/bbstored/boxbackup.redirectme.net-cert.pem
      gaga/bbstored/clientCA.pem

3) You may wish to read the configuration file
      gaga/bbstored.conf
   and adjust as appropraite.

4) Create accounts with bbstoreaccounts

5) Start the backup store daemon with the command
      /usr/local/sbin/bbstored gaga/bbstored.conf
   in /etc/rc.local, or your local equivalent.

===================================================================
```


For each account
```bash
mkdir 00000001
```

```bash
cd boxbackup-arm
docker-compose up -d --build
mkdir ca
chmod 777 ca
docker run -it -v $(pwd)/ca/:/ca/ boxbackupxarm_caserver
bbstored-certs ca init
```


```bash
cd boxbackup-x64
docker-compose up -d --build
mkdir ca
chmod 777 ca
docker run -it -v $(pwd)/ca/:/ca/ boxbackupx64_caserver
# since the ca directory is mounted into the container we must prevent the script from creating it
sed -i.bak 's/mkdir\(\$cert_dir,0700\)\n\s+&&\s//' /usr/bin/bbstored-certs
# since the regex doesn't work remove the first mkdir on line 94 with vi
vi /usr/bin/bbstored-certs
bbstored-certs ca2 init
```
