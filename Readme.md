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

Using the able `docker run` command with the `--restart=always` option will tell
Docker to always start our daemon container when Docker comes up. You need
to make sure the host starts Docker.


