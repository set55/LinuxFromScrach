# Linux from scratch in ubuntu

## Host System Requirement

version_check.sh to check requirement
```bash
bash version_check.sh
```

## **Creating a New Partition**

```bash
mkfs -v -t ext4 /dev/<xxx>
```

if mkfs fail, maybe the /dev/<xxx> is mounted, you should use command:
```bash
sudo umount -l /dev/sda1
```
-l can force umount it even some process using the drive

## **Setting The $LFS Variable**

Add export LFS=/mnt/lfs to /etc/environment. make sure all user(include root) can read $LFS
```bash
export LFS=/mnt/lfs
source /etc/environment
```

## **Mounting the New Partition**

Create $LFS dir and mount

```bash
mkdir -pv $LFS
mount -v -t ext4 /dev/<xxx> $LFS
```

Create sources in $LFS to obtain all the necessary packages and patches to build LFS

```bash
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources
```

download source file list(https://www.linuxfromscratch.org/lfs/view/stable/wget-list-sysv)

```bash
wget -O wget-list-sysv https://www.linuxfromscratch.org/lfs/view/stable/wget-list-sysv
```

download source file to $LFS/sources

```bash
wget --input-file=wget-list-sysv --continue --directory-prefix=$LFS/sources
```

## **Creating a Limited Directory Layout in the LFS Filesystem**

Create the required directory layout by issuing the following commands as `*root*`:

```bash
bash creating-limited-dir.sh
```

Programs in [Chapter 6](https://www.linuxfromscratch.org/lfs/view/stable/chapter06/chapter06.html) will be compiled with a cross-compiler (more details can be found in section [Toolchain Technical Notes](https://www.linuxfromscratch.org/lfs/view/stable/partintro/toolchaintechnotes.html)). This cross-compiler will be installed in a special directory, to separate it from the other programs. Still acting as `*root*`, create that directory with this command:
```
mkdir -pv $LFS/tools
```

## Adding the LFS User
When logged in as user root, making a single mistake can damage or destroy a system. Therefore, the packages in the next two chapters are built as an unprivileged user. You could use your own user name, but to make it easier to set up a clean working environment, we will create a new user called lfs as a member of a new group (also named lfs) and run commands as lfs during the installation process. As root, issue the following commands to add the new user:

```
groupadd lfs

useradd -s /bin/bash -g lfs -m -k /dev/null lfs

passwd lfs(12345678)
```

Grant lfs full access to all the directories under $LFS by making lfs the owner:
```
bash grant-lfs.sh

su - lfs
```

## Setting Up the Environment
Set up a good working environment by creating two new startup files for the bash shell. While logged in as user lfs, issue the following command to create a new .bash_profile:
```
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```

The new instance of the shell is a non-login shell, which does not read, and execute, the contents of the /etc/profile or .bash_profile files, but rather reads, and executes, the .bashrc file instead. Create the .bashrc file now:
```
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
EOF
```

For many modern systems with multiple processors (or cores) the compilation time for a package can be reduced by performing a "parallel make" by telling the make program how many processors are available via a command line option or an environment variable. For instance, an Intel Core i9-13900K processor has 8 P (performance) cores and 16 E (efficiency) cores, and a P core can simultaneously run two threads so each P core are modeled as two logical cores by the Linux kernel. As the result there are 32 logical cores in total. One obvious way to use all these logical cores is allowing make to spawn up to 32 build jobs. This can be done by passing the -j32 option to make:
```
make -j32
```

Or set the MAKEFLAGS environment variable and its content will be automatically used by make as command line options:
```
export MAKEFLAGS=-j32
```

To use all logical cores available for building packages in Chapter 5 and Chapter 6, set MAKEFLAGS now in .bashrc:
```
cat >> ~/.bashrc << "EOF"
export MAKEFLAGS=-j$(nproc)
EOF
```

Finally, to ensure the environment is fully prepared for building the temporary tools, force the bash shell to read the new user profile
```
source ~/.bash_profile
```



