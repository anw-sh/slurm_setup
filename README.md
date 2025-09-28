# Setting up Slurm

OS: **Fedora Server 42**

Cluster details:

| Node <br> (hostname) | Role | IP |
|------|------|----|
| access | Master/Login | 10.10.10.1 |
| atlas | Storage | 10.10.10.2 |
| pulse | GPU 1 | 10.10.10.3 |
| lyra | CPU 1 | 10.10.10.4 |
| nebula | CPU 2 | 10.10.10.5 |
| orion | CPU 3 | 10.10.10.6 |
| sirius | CPU 4 | 10.10.10.7 |
| titan | GPU 2 | 10.10.10.8 |

`/etc/hosts` file on each node should include all the IPs and hostnames, either in full or short format
```bash
10.10.10.1  access
# ... upto ...
10.10.10.8  titan
```

> Set password-less SSH between the nodes for easier file transfer  
> Run all commands as `root`

## Munge
Authentication service.

### On *access*
Create user account for `munge`  

```bash
$ export MUNGEUSER=1002
$ groupadd -g $MUNGEUSER munge
$ useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge
```

Download packages
```bash
$ dnf install munge munge-libs munge-devel
```
Create munge key
```bash
$ mungekey
```

Make directories, change ownership and set permissions
```bash
$ mkdir /run/munge
$ chown -R munge: /etc/munge /var/log/munge /var/lib/munge /run/munge
$ chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
```

Enable and start the service
```bash
$ systemctl enable munge.service
$ systemctl start munge.service
$ systemctl status munge.service
```

Check status on _Access_
```bash
$ munge -n | unmunge
# One of the lines is 
# STATUS:          Success (0)

# After installing on other nodes - check the status from access node
$ munge -n | ssh 10.10.10.2 unmunge
```

### On other nodes
Create `munge` account  

> Use the same UID and GID (REQUIRED)

```bash
$ export MUNGEUSER=1002
$ groupadd -g $MUNGEUSER munge
$ useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge
```

Install, copy key, create directories, change ownership, and set permissions  
```bash
$ dnf install munge munge-libs munge-devel
# Copy key
$ scp 10.10.10.1:/etc/munge/munge.key /etc/munge/
$ mkdir /run/munge
$ chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
$ chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
```
Enable and start `munge.service`  
Check the status as mentioned in the previous section

## Mariadb or MySQL
Install either of these - [RESOURCE](https://docs.fedoraproject.org/en-US/quick-docs/installing-mysql-mariadb/)

### On *access*
```bash
$ dnf install mariadb-server
$ systemctl enable mariadb
$ systemctl start mariadb
$ systemctl status mariadb
```
Configure with defaults and set `root` password, if required.   
```bash
$ mysql_secure_installation
```

### On other nodes
Install and setup like above  
Use the same password on other nodes as well.


## Slurm
The job scheduler

### On *access*
Create `slurm` user account
```bash
$ export SLURMUSER=1003
$ groupadd -g $SLURMUSER slurm
$ useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm
```

Install required dependencies to build and install SLURM RPMs
```bash
$ dnf install dbus-devel libbpf-devel infiniband-diags openssl openssl-devel pam-devel rpmbuild numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel man2html libibmad libibumad gcc autoconf automake make mariadb-devel perl perl-devel curl libcurl-devel
```

Download slurm package from [HERE](https://www.schedmd.com/download-slurm/)
```bash
# Latest version at the time of writing this document
$ wget https://download.schedmd.com/slurm/slurm-25.05.3.tar.bz2
```

Build RPMs
```bash
$ rpmbuild -ta slurm-25.05.3.tar.bz2
```

This generated RPMs but with warnings. 

>  Tried the following as suggested in this [thread](https://groups.google.com/g/slurm-users/c/W8YfGIn1rDI)
> ```bash
> $ rpmbuild -v -ta --define "_lto_cflags %{nil}" slurm-22.05.2.tar.bz2
> ```
>
> Still showing the same warnings.

End of the build process looks like this 
```bash
Provides: slurm-pam_slurm = 25.05.3-1.fc42 slurm-pam_slurm(x86-64) = 25.05.3-1.fc42
Requires(rpmlib): rpmlib(CompressedFileNames) <= 3.0.4-1 rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1
Requires: libc.so.6()(64bit) libc.so.6(GLIBC_2.14)(64bit) libc.so.6(GLIBC_2.2.5)(64bit) libc.so.6(GLIBC_2.3.4)(64bit) libc.so.6(GLIBC_2.33)(64bit) libc.so.6(GLIBC_2.34)(64bit) libc.so.6(GLIBC_2.38)(64bit) libc.so.6(GLIBC_2.4)(64bit) libc.so.6(GLIBC_ABI_DT_RELR)(64bit) libm.so.6()(64bit) libresolv.so.2()(64bit) libslurm.so.43()(64bit) rtld(GNU_HASH)
Obsoletes: pam_slurm <= 25.05.3
Checking for unpackaged file(s): /usr/lib/rpm/check-files /root/rpmbuild/BUILD/slurm-25.05.3-build/BUILDROOT
Wrote: /root/rpmbuild/SRPMS/slurm-25.05.3-1.fc42.src.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-example-configs-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-contribs-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-openlava-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-sackd-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-torque-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-pam_slurm-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-devel-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-libpmi-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-slurmdbd-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-slurmd-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-perlapi-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-slurmctld-25.05.3-1.fc42.x86_64.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/slurm-25.05.3-1.fc42.x86_64.rpm
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.9PgyEY
+ umask 022
+ cd /root/rpmbuild/BUILD/slurm-25.05.3-build
+ cd slurm-25.05.3
+ rm -rf /root/rpmbuild/BUILD/slurm-25.05.3-build/BUILDROOT
+ RPM_EC=0
++ jobs -p
+ exit 0
Executing(rmbuild): /bin/sh -e /var/tmp/rpm-tmp.hMYVP2
+ umask 022
+ cd /root/rpmbuild/BUILD/slurm-25.05.3-build
+ test -d /root/rpmbuild/BUILD/slurm-25.05.3-build
+ /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w /root/rpmbuild/BUILD/slurm-25.05.3-build
+ rm -rf /root/rpmbuild/BUILD/slurm-25.05.3-build
+ RPM_EC=0
++ jobs -p
+ exit 0

RPM build warnings:
    Macro expanded in comment on line 31: %_prefix path		install path for commands, libraries, etc.

    Macro expanded in comment on line 244: %define _unpackaged_files_terminate_build      0

    /root/rpmbuild/SPECS/slurm.spec line 412: autopatch: no matching patches in range
    %source_date_epoch_from_changelog is set, but %changelog has no entries to take a date from
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
    Deprecated external dependency generator is used!
```

Proceeding with the built RPMs
```bash
$ cd rpmbuild/RPMS/x86_64
$ rpm -i slurm-*
```

#### `slurm_acct_db`
Setup a separate DB for slurm
```bash
$ mysql -u root -p
```
Check hostname, generate user with password, grant permissions, and create a DB
```sql
SELECT @@hostname;
CREATE USER 'slurm'@'access' IDENTIFIED BY 'slurm_password';
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'access' WITH GRANT OPTION;
CREATE DATABASE slurm_acct_db;
```

#### Directories
`/etc/slurm` directory gets generated after installing slurm RPMs.  
Make `/var/lib/slurm`, `/var/log/slurm`, `/run/slurm` dirs, set ownership and permissions

```bash
$ mkdir /var/lib/slurm /var/log/slurm /run/slurm
$ chown -R slurm: /var/log/slurm /var/lib/slurm /etc/slurm /run/slurm
$ chmod 0700 /var/log/slurm /var/lib/slurm /run/slurm
$ chmod 755 /etc/slurm
```

Additional directories & files
```bash
$ mkdir /var/spool/slurmctld
$ chown slurm:slurm /var/spool/slurmctld
$ chmod 755 /var/spool/slurmctld
$ touch /var/log/slurm/slurm_jobacct.log /var/log/slurm/slurm_jobcomp.log
$ chown -R slurm:slurm /var/log/slurm
```
Run `slurmd -C` to get node information (Can be used for filling `slurm.conf` - Not required on master node)

#### Configuration and services
Generate `slurmdbd.conf`, `slurm.conf`, `cgroup.conf` with required options in `/etc/slurm`.  
Config files of my cluster were provided for reference. Modify as required

After modifying...  
```bash
$ systemctl enable slurmdbd.service
$ systemctl start slurmdbd.service
$ systemctl status slurmdbd.service
$ systemctl enable slurmctld.service
$ systemctl start slurmctld.service
$ systemctl status slurmctld.service
```
Check the output of `systemctl status` for errors and troubleshoot accordingly, if any

Copy `cgroup.conf`, `slurm.conf` and `slurmdbd.conf` files to the worker nodes after setting up slurm.


### On other nodes
Create `slurm` user account.   

> Use the same UID and GID as *access* (REQUIRED)  

```bash
$ export SLURMUSER=1003
$ groupadd -g $SLURMUSER slurm
$ useradd  -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm
```

Copy the RPMs and install on other nodes  
```bash
$ rsync -avrP 10.8.32.250:/root/rpmbuild ./
```

> Are the dependencies required? Tried installing without dependencies - did not work  
> Dependencies are required (may be not all). Anyway install all those here as well.  

After that...
```bash
$ cd rpmbuild/RPMS/x86_64/
$ rpm -i slurm-*
```

#### Directories
```bash
$ mkdir /var/spool/slurmd
$ chown slurm: /var/spool/slurmd
$ chmod 755 /var/spool/slurmd
$ mkdir /var/log/slurm
$ touch /var/log/slurm/slurmd.log
$ chown -R slurm:slurm /var/log/slurm/slurmd.log
```
Run `slurmd -C` to get node information


Add each node's information in the *access*'s `slurm.conf` and copy it to all nodes  
Enable and start `slurmd.service` (**NOT `slurmdbd`**). Check `status` for any errors  
Restart `slurmctld` on *access*

### On GPU nodes
Add `gres.conf` file with Graphics device information in `/etc/slurm`.   
Output of `slurmd -C` has Graphics device information as well   
*titan*'s `gres.conf` provided.   
NVIDIA drivers should be installed prior to slurm setup.

***

Start/Restart `slurmd` service on worker nodes  
Restart `slurmdbd` and `slurmctld` services on *access* node 

Run `sinfo`  
All nodes should be `UP` and `idle`, except *atlas*, which was intentionally set to `DOWN` in my cluster.  
Run test scripts

***

## Troubleshooting
Issues I faced while setting up slurm

1. `cgroup_v2.so` missing  
   Building RPMs may require additional packages like `libbpf-devel`, `dbus-devel`, etc.  
   The RPMs built without such dependencies are missing `cgroup_v2.so` in `/usr/lib64/slurm`.  
   
   So, uninstalled and removed the built RPMs, and then installed the required dependencies  
   Then the RPMs were rebuilt. After installation `cgroup_v2.so` was generated

   ```bash
   $ dnf remove slurm*
   $ rm -rvI rpmbuild/RPMS/x86_64/* rpmbuild/SRPMS/slurm-25.05.0-1.fc42.src.rpm
   # These were added in the 'install dependencies' step
   $ dnf install dbus-devel libbpf-devel infiniband-diags mariadb-server
   ```
   May require to drop the old database and make a new one  

   Started `slurmd.service` on nodes  
   ```bash
   $ systemctl enable slurmd.service
   $ systemctl start slurmd.service
   ```

   Succeded on _nebula_, _orion_, _lyra_   
   Restarted `slurmctld.service` on _access_

2. MySQL database settings  
   
   `error: Database settings not recommended values: innodb_buffer_pool_size innodb_lock_wait_timeout`  
   
   Check the default values (Enter `mysql` terminal on *access* - `mysql -u root -p`)
   ```sql
   > SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
   # innodb_buffer_pool_size | 134217728
   > SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
   # innodb_lock_wait_timeout | 50 
   ```
   Changed these values in `/etc/my.cnf` as suggested in the [Documentation](https://slurm.schedmd.com/accounting.html#slurm-accounting-configuration-before-build)  
   Restarted services on *access* - `mariadb`, `slurmdbd`, `slurmctld`

3. Another issue was, non-root users were unable to use `sinfo`
   ```bash
   $ sinfo
   # sinfo: error: resolve_ctls_from_dns_srv: res_nsearch error: Unknown host
   # sinfo: error: fetch_config: DNS SRV lookup failed
   # sinfo: error: _establish_config_source: failed to fetch config
   # sinfo: fatal: Could not establish a configuration source
   ```

   This was due to the permissions on `/etc/slurm` directory  
   Changed to `755` - check the "Directories" section above

4. _pulse_ and *titan* required additional configuration file - `gres.conf`  
   GPU node is working but showing it's state as `drain` - Because of direct tasks that are already running  
   Tried Resuming the node - `scontrol update NodeName="pulse" State="RESUME"` - _pulse_ is working now. Same for *titan*.

5. _lyra_ is not staying `UP`
   Restarted `munge` service - Now the node is `UP`  
   Still there seems to be a problem  
   It was related to firewall - Firewall was turned off and now _lyra_ is taking jobs

*** 

## Uninstall
- Stop and disable all the above services in the corresponding nodes
- Remove DB and User created in MySQL
- Uninstall the `slurm` and `munge` packages

## Reference
https://southgreenplatform.github.io/trainings/hpc/slurminstallation/

--- END ---
