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
#docker-compose up -d --build
docker-compose build

mkdir usb-disk
mkdir ca
chmod 777 ca

docker run -it --hostname boxbackup -v $(pwd)/ca/:/ca/ -v $(pwd)/usb-disk/:/boxbackup-data/ --rm=false boxbackupx64_bbtempserver
bbstored-config /etc/boxbackup 0.0.0.0 boxbackup
cp /etc/boxbackup/bbstored/0.0.0.0-csr.pem /boxbackup-data/
cp /etc/boxbackup/bbstored/0.0.0.0-csr.pem /ca/
cp /etc/boxbackup/bbstored/0.0.0.0-key.pem /boxbackup-data/

bbstoreaccounts create 1 0 900G 920G
bbstoreaccounts create 2 0 300G 350G
bbstoreaccounts create 3 0 300G 350G
bbstoreaccounts create 4 0 300G 350G
bbstoreaccounts create 5 0 300G 350G
bbstoreaccounts create 6 0 300G 350G
bbstoreaccounts create 7 0 300G 350G
cp /etc/boxbackup/bbstored/accounts.txt /boxbackup-data/

# Pretend we are the client
bbackupd-config /etc/box lazy 1 boxbackup.redirectme.net /var/bbackupd /home
mdkir /ca/client-1
cp /etc/box/bbackupd/1-FileEncKeys.raw /ca/client-1/
cp /etc/box/bbackupd/1-csr.pem /ca/client-1/

bbackupd-config /etc/box lazy 2 boxbackup.redirectme.net /var/bbackupd /home
mdkir /ca/client-2
cp /etc/box/bbackupd/2-FileEncKeys.raw /ca/client-2/
cp /etc/box/bbackupd/2-csr.pem /ca/client-2/

bbackupd-config /etc/box lazy 3 boxbackup.redirectme.net /var/bbackupd /home
mdkir /ca/client-3
cp /etc/box/bbackupd/3-FileEncKeys.raw /ca/client-3/
cp /etc/box/bbackupd/3-csr.pem /ca/client-3/

bbackupd-config /etc/box lazy 4 boxbackup.redirectme.net /var/bbackupd /home
mdkir /ca/client-4
cp /etc/box/bbackupd/4-FileEncKeys.raw /ca/client-4/
cp /etc/box/bbackupd/4-csr.pem /ca/client-4/

bbackupd-config /etc/box lazy 5 boxbackup.redirectme.net /var/bbackupd /home
mdkir /ca/client-5
cp /etc/box/bbackupd/5-FileEncKeys.raw /ca/client-5/
cp /etc/box/bbackupd/5-csr.pem /ca/client-5/

bbackupd-config /etc/box lazy 6 boxbackup.redirectme.net /var/bbackupd /home
mdkir /ca/client-6
cp /etc/box/bbackupd/6-FileEncKeys.raw /ca/client-6/
cp /etc/box/bbackupd/6-csr.pem /ca/client-6/

bbackupd-config /etc/box lazy 7 boxbackup.redirectme.net /var/bbackupd /home
mdkir /ca/client-7
cp /etc/box/bbackupd/7-FileEncKeys.raw /ca/client-7/
cp /etc/box/bbackupd/7-csr.pem /ca/client-7/

exit

# back on the host command line
#cd boxbackup-x64


docker run -it -v $(pwd)/ca/:/ca/ boxbackupx64_caserver
bbstored-certs catmp init
mv /catmp/* /ca/
chmod -R a+rwx /ca

bbstored-certs ca sign-server ca/0.0.0.0-csr.pem
# yes
exit

# back on the host command line
cp ca/servers/0.0.0.0-cert.pem usb-disk/0.0.0.0-cert.pem
cp ca/roots/clientCA.pem usb-disk/clientCA.pem

bbstored-certs ca sign 1-csr.pem
bbstored-certs ca sign 2-csr.pem
bbstored-certs ca sign 3-csr.pem
bbstored-certs ca sign 4-csr.pem
bbstored-certs ca sign 5-csr.pem
bbstored-certs ca sign 6-csr.pem
bbstored-certs ca sign 7-csr.pem

# start the server container
docker run -d --restart=always --hostname boxbackup -v $(pwd)/usb-disk/:/boxbackup-data/ -p 2201:2201 --rm=false boxbackupx64_bbserver



```
===================================================================

bbackupd basic configuration complete.

What you need to do now...

1) Make a backup of /etc/box/bbackupd/1-FileEncKeys.raw
   This should be a secure offsite backup.
   Without it, you cannot restore backups. Everything else can
   be replaced. But this cannot.
   KEEP IT IN A SAFE PLACE, OTHERWISE YOUR BACKUPS ARE USELESS.

2) Send /etc/box/bbackupd/1-csr.pem
   to the administrator of the backup server, and ask for it to
   be signed.

3) The administrator will send you two files. Install them as
      /etc/box/bbackupd/1-cert.pem
      /etc/box/bbackupd/serverCA.pem
   after checking their authenticity.

4) You may wish to read the configuration file
      /etc/box/bbackupd.conf
   and adjust as appropriate.
   
   There are some notes in it on excluding files you do not
   wish to be backed up.

5) Review the script
      /etc/box/bbackupd/NotifySysadmin.sh
   and check that it will email the right person when the store
   becomes full. This is important -- when the store is full, no
   more files will be backed up. You want to know about this.

6) Start the backup daemon with the command
      /usr/local/sbin/bbackupd /etc/box/bbackupd.conf
   in /etc/rc.local, or your local equivalent.
   Note that bbackupd must run as root.

===================================================================





1) Sign /etc/boxbackup/bbstored/boxbackup.redirectme.net-csr.pem
   using the bbstored-certs utility.

2) Install the server certificate and root CA certificate as
      /etc/boxbackup/bbstored/boxbackup.redirectme.net-cert.pem
      /etc/boxbackup/bbstored/clientCA.pem

3) You may wish to read the configuration file
      /etc/boxbackup/bbstored.conf
   and adjust as appropraite.

4) Create accounts with bbstoreaccounts

5) Start the backup store daemon with the command
      /usr/local/sbin/bbstored
   in /etc/rc.local, or your local equivalent.


        CertificateFile = /etc/boxbackup/bbstored/boxbackup.redirectme.net-cert.pem
        PrivateKeyFile = /etc/boxbackup/bbstored/boxbackup.redirectme.net-key.pem
        TrustedCAsFile = /etc/boxbackup/bbstored/clientCA.pem
