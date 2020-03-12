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
	boxbackuparm_bbserver
```

## Implementation

The official BoxBackup documentation suggests that you set up a server to sign the
certificates. This should be separate from the actual backup server. And the clients
also have certificates to encrypt and decrypt the files and be allowed to backup.

I have here taken the view that, as the family IT guy, I need to sign the certs and
provide most of installation. Thus I need one docker machine that can do all the signing
and account stuff and one conatainer that runs the backup server.

The first one needs the ca directory where all the certs go plus the boxbackup-data directory
where the stuff goes that the backup server needs to run plus all the backup data. These 
directories are mounted into the containers as volumes so that the containers are stateless
and can be upgrades, changed etc.

If this doesn't match your use-case, review the implementation anyway and adjust as necessary.

## Setting it all up

```bash
# if you are on an Intel processor
cd boxbackup-x64
# or if you are on a Raspberry Pi
cd boxbackup-arm

docker-compose build

# create the two directories. You can move the boxbackup-data to wherever you have space afterwards
mkdir -p boxbackup-data ca
chmod 777 boxbackup-data ca

# On Intel
docker run -it --hostname boxbackup -v $(pwd)/ca/:/ca/ -v $(pwd)/boxbackup-data/:/boxbackup-data/ --rm=false boxbackupx64_bbtempserver
# On Rapsberry Pi
docker run -it --hostname boxbackup -v $(pwd)/ca/:/ca/ -v $(pwd)/boxbackup-data/:/boxbackup-data/ --rm=false boxbackuparm_bbtempserver

# create server keys
bbstored-config /etc/boxbackup 0.0.0.0 boxbackup
cp /etc/boxbackup/bbstored/0.0.0.0-csr.pem /boxbackup-data/
cp /etc/boxbackup/bbstored/0.0.0.0-csr.pem /ca/
cp /etc/boxbackup/bbstored/0.0.0.0-key.pem /boxbackup-data/
chmod a+rwx /boxbackup-data/0.0.0.0-key.pem

# create 7 backup accounts with soft and hard limits
bbstoreaccounts create 1 0 900G 920G
bbstoreaccounts create 2 0 300G 350G
bbstoreaccounts create 3 0 300G 350G
bbstoreaccounts create 4 0 300G 350G
bbstoreaccounts create 5 0 300G 350G
bbstoreaccounts create 6 0 300G 350G
bbstoreaccounts create 7 0 300G 350G
cp /etc/boxbackup/bbstored/accounts.txt /boxbackup-data/

# In an ideal world the client would create these keys and you would transfer
# them to the CA server, sign them there and send the signed keys back.
# Here I am simply generating it all on the temporary server
bbackupd-config /etc/box lazy 1 hostname.com /var/bbackupd /home
mkdir /ca/client-1
cp /etc/box/bbackupd/1-FileEncKeys.raw /ca/client-1/
cp /etc/box/bbackupd/1-csr.pem /ca/client-1/
cp /etc/box/bbackupd/1-key.pem /ca/client-1/

bbackupd-config /etc/box lazy 2 hostname.com /var/bbackupd /home
mkdir /ca/client-2
cp /etc/box/bbackupd/2-FileEncKeys.raw /ca/client-2/
cp /etc/box/bbackupd/2-csr.pem /ca/client-2/
cp /etc/box/bbackupd/2-key.pem /ca/client-2/

bbackupd-config /etc/box lazy 3 hostname.com /var/bbackupd /home
mkdir /ca/client-3
cp /etc/box/bbackupd/3-FileEncKeys.raw /ca/client-3/
cp /etc/box/bbackupd/3-csr.pem /ca/client-3/
cp /etc/box/bbackupd/3-key.pem /ca/client-3/

bbackupd-config /etc/box lazy 4 hostname.com /var/bbackupd /home
mkdir /ca/client-4
cp /etc/box/bbackupd/4-FileEncKeys.raw /ca/client-4/
cp /etc/box/bbackupd/4-csr.pem /ca/client-4/
cp /etc/box/bbackupd/4-key.pem /ca/client-4/

bbackupd-config /etc/box lazy 5 hostname.com /var/bbackupd /home
mkdir /ca/client-5
cp /etc/box/bbackupd/5-FileEncKeys.raw /ca/client-5/
cp /etc/box/bbackupd/5-csr.pem /ca/client-5/
cp /etc/box/bbackupd/5-key.pem /ca/client-5/

bbackupd-config /etc/box lazy 6 hostname.com /var/bbackupd /home
mkdir /ca/client-6
cp /etc/box/bbackupd/6-FileEncKeys.raw /ca/client-6/
cp /etc/box/bbackupd/6-csr.pem /ca/client-6/
cp /etc/box/bbackupd/6-key.pem /ca/client-6/

bbackupd-config /etc/box lazy 7 hostname.com /var/bbackupd /home
mkdir /ca/client-7
cp /etc/box/bbackupd/7-FileEncKeys.raw /ca/client-7/
cp /etc/box/bbackupd/7-csr.pem /ca/client-7/
cp /etc/box/bbackupd/7-key.pem /ca/client-7/

# Sign the certificates (which you would do on the CA server)

# if you already have the ca directory with all the certs, skip this
# else create all the CA certs here:
bbstored-certs catmp init
mv /catmp/* /ca/
chmod -R a+rwx /ca
echo yes | bbstored-certs ca sign-server /ca/0.0.0.0-csr.pem

# Copy the root certs to the boxbackup-data volume for the server to use
cp /ca/servers/0.0.0.0-cert.pem /boxbackup-data/0.0.0.0-cert.pem
cp /ca/roots/clientCA.pem /boxbackup-data/clientCA.pem

# Sign the client cert requests
echo yes | bbstored-certs ca sign /ca/client-1/1-csr.pem
echo yes | bbstored-certs ca sign /ca/client-2/2-csr.pem
echo yes | bbstored-certs ca sign /ca/client-3/3-csr.pem
echo yes | bbstored-certs ca sign /ca/client-4/4-csr.pem
echo yes | bbstored-certs ca sign /ca/client-5/5-csr.pem
echo yes | bbstored-certs ca sign /ca/client-6/6-csr.pem
echo yes | bbstored-certs ca sign /ca/client-7/7-csr.pem
exit

# This command starts the backup server and mounts the boxbackup-data directory to it.
# be sure to specify the absolute path of the directory. The directoy can be anywhere 
# your host can see it.
# For Intel:
docker run -d --restart=always --hostname boxbackup -v $(pwd)/boxbackup-data/:/boxbackup-data/ -p 2201:2201 --rm=false boxbackupx64_bbserver
# For Raspberry Pi
docker run -d --restart=always --hostname boxbackup -v /usb-disk/boxbackup-data/:/boxbackup-data/ -p 2201:2201 --rm=false boxbackuparm_bbserver

# If you need to check something inside the running server:
docker exec -it $(docker ps | grep bbserver | cut -d' ' -f1) bash

# check the logfile
less boxbackup-data/logfile


# for each of the clients you need the following files in the bbackupd config directory and the bbackupd.config needs to refer to these.
# Follow the official instructions for setting up the client agent for Windows or Linux
cp ca/roots/serverCA.pem /etc/boxbackup/bbackupd/
cp ca/clients/1-cert.pem /etc/boxbackup/bbackupd/
cp ca/client-1/1-FileEncKeys.raw /etc/boxbackup/bbackupd/
cp ca/client-1/1-key.pem /etc/boxbackup/bbackupd/
cp ca/client-1/1-csr.pem /etc/boxbackup/bbackupd/

# keep the ca directory in a safe place
```