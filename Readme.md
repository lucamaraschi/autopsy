# Autopsy

dissect dead node service core dumps with mdb via a smart os vm

## Why?

mdb is an awesome debugger that comes with smart os.

There's an extension for it that allows you to 
postmortem node core files

However, a tremendous amount of people run node
on linux (not least because of docker and what not). 

But it turns out that mdb can analyse linux core files,
we just have to give it the core file and the node binary
that was running when the core file was generated.

The problem is, actually getting you linux core file
into an environment that is running a version of mdb 
that this can work with is... painful.

So, autopsy abstracts that pain away, and installs an `autopsy`
executable on linux that essentially acts as a proxy to the 
mdb client within the smartos vm.

You can also run autopsy on OS X, but you'll need the linux
node binary and core file to pass to it. 


## Setup

### Preqrequisites

Autopsy depends on virtual box, on ubuntu/debian we can do

```sh
sudo apt-get install virtualbox
```

Sometimes the virtualbox package for ubuntu seems to have
problems with kernel versions (particularly if we're installing
virtualbox into an ubuntu vm). We can check whether the virtualbox
install was successful with: 

```sh
VBoxManage --version
```

If we see some error output about the character device /dev/vboxdrv 
does not exist then there's a problem. 


In this case we can grab the latest `.deb` file from the virtualbox site, e.g.:

```sh
$ sudo apt-get purge virtualbox
$ curl http://download.virtualbox.org/virtualbox/5.0.10/virtualbox-5.0_5.0.10-104061~Ubuntu~trusty_amd64.deb > vbox.deb
$ sudo apt-get install linux-headers-`uname -r`
$ sudo dpkg -i vbox.deb
```

To find the right deb package for your Linux distro see <https://www.virtualbox.org/wiki/Linux_Downloads>. 

There's also currently a hard dependency on `expect`

```sh
sudo apt-get install expect
```

On OS X - well we can work it out ;)

### Install

```
sudo npm install -g autopsy
```

This will install autopsy on the system, download smartos
virtual machine assets and setup a smartos vm in virtual box. 

The VM assets download is ~450mb, in testing on a fairly decent
connection, setup from start to finish (not including npm dep installs)
takes around 1.5 minutes. This is because we're using multithreaded 
downloading and host the assets on S3. 

Once finished the following executables will be available

* autopsy-start - starts the vm
* autopsy-stop - stops the vm
* autopsy-status - gets vm status
* autopsy-setup - runs setup (only needful for troubleshooting)
* autopsy - provides the CLI proxy to mdb in the vm

### Resuming Setup

If postinstall setup is interupted for any reason (including network failure
during assets download), try again with

```sh
autopsy-setup
```

If there was a partial download, it should resume rather than restart. 

## autopsy

The autopsy command takes the following args

```sh
autopsy [node-binary] core-file
```

On OS X the node binary is not optional, on linux
if not supplied the current installed node binary
will be used. 

When this command is run the following occurs

1. copy the node binary and core file into the vm (in a purpose built smartos zone)
2. ssh into the vm, login to the relevant smartos zone
3. run mdb with the copied core dump and node binary files
4. inject `::load v8` to get the v8 related debugging commands
5. pipe host machine process.stdin to the mdb interactive environment and pipe the mdb stdout to the host machine stdout

For using mdb see the [mdb reference docs][]


## Generating a core file

In production, if we run our node processes with `--abort-on-uncaught-exception` we will always get a core dump when a process crashes (that is,
as long as our linux environment is set up correctly)

You can also manually generate a core file using `process.abort()`.

Finally a core file can also be obtained by attaching `gdb` to a running processing and executing `generate-core`. 

## Setting up Linux to generate core files

If you're using an ubuntu server (and probably debian etc. etc.) you may have apport installed - this intercepts core files so we need to get rid of it

```
sudo apt-get purge apport
```

Next you need to make sure that linux is configured to allocate
space for the core file, like so

```
ulimit -c unlimited
```

Put this in a start up script and what not. 

## Why's it so big?

We'll get it smaller (hopefully), this is a first pass
and we're focusing on functionality. But the size it's 
because we're like.. running an entire virtual machine. 

## Example

The `example` folder has a `core` and `node` file that we're
generated by the `die.js` file 

You can try out autopsy with these two files (on OS X and 
Linux), from the same folder as this readme do

```
autopsy example/node example/core 
```

Once the mdb console appears you can try 

```
> ::jsstack
```

For starters, and then if you want to get fancy 

```
> ::findjsobjects -p myproperty
137289672551
> 137289672551::jsprint
```

## todo

* get up arrow (for history) working

## Future

* reduce size of everything
* maybe work with saved machine state to decrease boot time
* can we get rid of the dep on virtual box I wonder?
* work out if this is a problem "mdb: warning: librtld_db failed to initialize; shared library information will not be available" - may need a patch in the vm
* get ssh keys into the vm instead of passing a pw with expect, granted the pw being plain text in this case doesn't matter, but we might be able to get rid of the dependency on expect if we get rid of the pw. 
* vm shouldn't need 5gb of ram, sort that out. 
* extend this into another project that runs node processes in smart os zones (just like docker containers) - but with the added benefit of native smart os core dumps which can be analysed with mdb



## mdb_v8 upgrade notes

The latest smartos comes with an old version of mdb_v8, as of autopsy 0.0.2 the vm runs mdb_v8 1.2.2 (latest at time of writing)
To upgrade the v8 version (without waiting for an autopsy release) we can perform the following steps

1. login to the vm `ssh -p 2222 root@localhost` pw: mdb
2. login to the zone `zlogin 7f3ba160-047c-4557-9e87-8157db23f205`
3. `mkdir /mdb && cd /mdb`
4. `pkgin install gcc49-4.9.1 gmake-4.0 git`
5. `git clone https://github.com/joyent/mdb_v8`
6. `cd mdb_v8 && make`
7. `cp mdb_v8/builds/amd64/mdb_v8.so .`

At this point we have successfully upgraded to latest mdb_v8, however 
we have a lot of extra dev packages installed in the vm making it much 
less lean. So, we may want to copy the `mdb_v8.so` file from the vm, like so:

```sh
scp -P 2222 root@localhost:/zones/7f3ba160-047c-4557-9e87-8157db23f205/root/mdb/mdb_v8.so .
```

Then recreate the vm (follow removing the vm below, then `npm run setup`) and copy the file back in (this is what we do for releases).

```sh
scp -P 2222 mdb_v8.so root@localhost:/zones/7f3ba160-047c-4557-9e87-8157db23f205/root/mdb
```


## removing the vm

Currently there's no command for removing the vm, follow these steps, in order

1. open virtual box, right click the vm, click remove - then click "delete all files"
2. rm the `assets` folder from the autopsy module folder `rm $(npm get prefix)/lib/node_modules/autopsy/assets`
3. make sure there isn't a smartos folder left in the virtual box virtual machines folder (`~/VirtualBox\ VMs`)


## Caveats

* We recommend installing globally, since there can only be one
smartos vm. 
* If the assets folder or any parent folder is moved or
renamed, the vm will fail to start (because it won't be able
to locate the the iso and vmdk files). In this case you would need to 
manually update virtual box with the paths.

[mdb reference docs]: https://github.com/joyent/mdb_v8/blob/master/docs/usage.md#node-specific-mdb-command-reference