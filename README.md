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

## Before Compilation Instructions

Check $LFS is /mnt/lfs
```
echo $LFS
```

The build instructions assume that the Host System Requirements, including symbolic links, have been set properly:
- bash is the shell in use.
- sh is a symbolic link to bash.
- /usr/bin/awk is a symbolic link to gawk.
- /usr/bin/yacc is a symbolic link to bison, or to a small script that executes bison.



1. Place all the sources and patches in a directory that will be accessible from the chroot environment, such as /mnt/lfs/sources/.

2. Change to the /mnt/lfs/sources/ directory.

3. For each package:

    - Using the **tar** program, extract the package to be built. In Chapter 5 and Chapter 6, ensure you are the lfs user when extracting the package.

    - Do not use any method except the tar command to extract the source code. Notably, using the **cp -R** command to copy the source code tree somewhere else can destroy timestamps in the source tree, and cause the build to fail.

    - Change to the directory created when the package was extracted.

    - Follow the instructions for building the package.

    - Change back to the sources directory when the build is complete.

    - Delete the extracted source directory unless instructed otherwise.


## Installation of Cross Binutils
change to sources and extract binutils-2.43.1.tar.xz
```
cd /mnt/lfs/sources
tar -xvf binutils-2.43.1.tar.xz
```

change to binutils-2.43.1, make build dir and change to build
```
cd binutils-2.43.1
mkdir -v build
cd       build
```

prepare compilation
```
time { ../configure --prefix=$LFS/tools \
             --with-sysroot=$LFS \
             --target=$LFS_TGT   \
             --disable-nls       \
             --enable-gprofng=no \
             --disable-werror    \
             --enable-new-dtags  \
             --enable-default-hash-style=gnu && make && make install; }
```

## Installation of Cross GCC
change to sources and extract
```
cd /mnt/lfs/sources
tar -xvf gcc-14.2.0.tar.xz
```

change to gcc-14.2.0, and extract others and 
```
cd gcc-14.2.0
tar -xf ../mpfr-4.2.1.tar.xz
mv -v mpfr-4.2.1 mpfr
tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp
tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc
```

On x86_64 hosts, set the default directory name for 64-bit libraries to “lib”:
```
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
 ;;
esac
```

make build dir and change to build
```
mkdir -v build
cd       build
```

prepare compilation
```
time { ../configure                  \
    --target=$LFS_TGT         \
    --prefix=$LFS/tools       \
    --with-glibc-version=2.40 \
    --with-sysroot=$LFS       \
    --with-newlib             \
    --without-headers         \
    --enable-default-pie      \
    --enable-default-ssp      \
    --disable-nls             \
    --disable-shared          \
    --disable-multilib        \
    --disable-threads         \
    --disable-libatomic       \
    --disable-libgomp         \
    --disable-libquadmath     \
    --disable-libssp          \
    --disable-libvtv          \
    --disable-libstdcxx       \
    --enable-languages=c,c++ && make && make install; }
```

This build of GCC has installed a couple of internal system headers. Normally one of them, limits.h, would in turn include the corresponding system limits.h header, in this case, $LFS/usr/include/limits.h. However, at the time of this build of GCC $LFS/usr/include/limits.h does not exist, so the internal header that has just been installed is a partial, self-contained file and does not include the extended features of the system header. This is adequate for building Glibc, but the full internal header will be needed later. Create a full version of the internal header using a command that is identical to what the GCC build system does in normal circumstances:

```
cd ..
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/include/limits.h
```

## Linux-6.10.5 API Headers
change to sources and extract
```
cd /mnt/lfs/sources
tar -xvf linux-6.10.5.tar.xz
```

change to linux-6.10.5, and 
```
cd linux-6.10.5

# Make sure there are no stale files embedded in the package

make mrproper

# Now extract the user-visible kernel headers from the source. The recommended make target “headers_install” cannot be used, because it requires rsync, which may not be available. The headers are first placed in ./usr, then copied to the needed location.

make headers
find usr/include -type f ! -name '*.h' -delete
cp -rv usr/include $LFS/usr
```