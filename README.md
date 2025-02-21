# Linux from scratch in ubuntu

## Host System Requirement

version_check.sh to check requirement
```bash
bash version_check.sh
```
If sh is not link to bash:
```
sudo ln -s /bin/bash /bin/sh

# check it
ls -l /bin/sh
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


## Binutils-2.43.1 - Pass 1
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

finish and change to sources and delete binutils-2.43.1
```
cd ..
rm -rf binutils-2.43.1
```

## GCC-14.2.0 - Pass 1
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

finish and change to sources and delete gcc-14.2.0
```
cd ..
rm -rf gcc-14.2.0
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

finish and change to sources and delete linux-6.10.5
```
cd ..
rm -rf linux-6.10.5
```

## Glibc-2.40
The Glibc package contains the main C library. 
change to sources and extract
```
cd /mnt/lfs/sources
tar -xvf glibc-2.40.tar.xz
```

```
# First, create a symbolic link for LSB compliance. Additionally, for x86_64, create a compatibility symbolic link required for proper operation of the dynamic library loader:

case $(uname -m) in
    i?86)   ln -sfv ld-linux.so.2 $LFS/lib/ld-lsb.so.3
    ;;
    x86_64) ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64
            ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64/ld-lsb-x86-64.so.3
    ;;
esac

# Some of the Glibc programs use the non-FHS-compliant /var/db directory to store their runtime data. Apply the following patch to make such programs store their runtime data in the FHS-compliant locations:

patch -Np1 -i ../glibc-2.40-fhs-1.patch
```

make build dir and change to build
```
mkdir -v build
cd       build

# Ensure that the ldconfig and sln utilities are installed into /usr/sbin:
echo "rootsbindir=/usr/sbin" > configparms

# Next, prepare Glibc for compilation:
../configure                             \
      --prefix=/usr                      \
      --host=$LFS_TGT                    \
      --build=$(../scripts/config.guess) \
      --enable-kernel=4.19               \
      --with-headers=$LFS/usr/include    \
      --disable-nscd                     \
      libc_cv_slibdir=/usr/lib

# Compile the package:
make

# Install the package:
make DESTDIR=$LFS install
```
Need to final check
```
# Fix a hard coded path to the executable loader in the ldd script:
sed '/RTLDLIST=/s@/usr@@g' -i $LFS/usr/bin/ldd

# To perform a sanity check, run the following commands:
echo 'int main(){}' | $LFS_TGT-gcc -xc -
readelf -l a.out | grep ld-linux
```
```
# If everything is working correctly, there should be no errors, and the output of the last command will be of the form:

[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]

# If the output is not as shown above, or there is no output at all, then something is wrong. Investigate and retrace the steps to find out where the problem is and correct it. This issue must be resolved before continuing.
```

Once all is well, clean up the test file:
```
rm -v a.out
```
finish and change to sources and delete glibc-2.40
```
cd ..
rm -rf glibc-2.40
```


## Libstdc++ from GCC-14.2.0
change to sources and extract
```
cd /mnt/lfs/sources
tar -xvf gcc-14.2.0.tar.xz
cd gcc-14.2.0
```


make build dir and change to build
```
mkdir -v build
cd       build
```

Prepare, compile, install Libstdc++
```
# prepare compile
../libstdc++-v3/configure           \
    --host=$LFS_TGT                 \
    --build=$(../config.guess)      \
    --prefix=/usr                   \
    --disable-multilib              \
    --disable-nls                   \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/14.2.0

# compile
make

# install
make DESTDIR=$LFS install
```

Remove the libtool archive files because they are harmful for cross-compilation:
```
rm -v $LFS/usr/lib/lib{stdc++{,exp,fs},supc++}.la
```

finish and change to sources and delete gcc-14.2.0
```
cd ..
rm -rf gcc-14.2.0
```

## M4-1.4.19
In $LFS/sources
```
tar -xvf m4-1.4.19.tar.xz
cd m4-1.4.19
```

Prepare, compile and install
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rf m4-1.4.19
```

## Ncurses-6.5
```
tar -xvf ncurses-6.5.tar.gz
cd ncurses-6.5
```

ensure that gawk is found first during configuration:
```
sed -i s/mawk// configure
```

Then, run the following commands to build the “tic” program on the build host:
```
# line by line
mkdir build
pushd build
  ../configure
  make -C include
  make -C progs tic
popd
```

prepare compire
```
./configure --prefix=/usr                \
            --host=$LFS_TGT              \
            --build=$(./config.guess)    \
            --mandir=/usr/share/man      \
            --with-manpage-format=normal \
            --with-shared                \
            --without-normal             \
            --with-cxx-shared            \
            --without-debug              \
            --without-ada                \
            --disable-stripping
```

compile and install
```
make
make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install
ln -sv libncursesw.so $LFS/usr/lib/libncurses.so
sed -e 's/^#if.*XOPEN.*$/#if 1/' \
    -i $LFS/usr/include/curses.h
```

change to sources and delete
```
cd ..
rm -rf ncurses-6.5
```

## Bash-5.2.32
```
tar -xvf bash-5.2.32.tar.gz
cd bash-5.2.32
```

prepare, compile and install
```
./configure --prefix=/usr                      \
            --build=$(sh support/config.guess) \
            --host=$LFS_TGT                    \
            --without-bash-malloc              \
            bash_cv_strtold_broken=no

make

make DESTDIR=$LFS install

ln -sv bash $LFS/bin/sh
```

change to sources and delete
```
cd ..
rm -rf bash-5.2.32
```

## Coreutils-9.5
```
tar -xvf coreutils-9.5.tar.xz
cd coreutils-9.5
```

prepare, compile and install
```
./configure --prefix=/usr                     \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --enable-install-program=hostname \
            --enable-no-install-program=kill,uptime

make

make DESTDIR=$LFS install
```

Move programs to their final expected locations. Although this is not necessary in this temporary environment, we must do so because some programs hardcode executable locations:
```
mv -v $LFS/usr/bin/chroot              $LFS/usr/sbin
mkdir -pv $LFS/usr/share/man/man8
mv -v $LFS/usr/share/man/man1/chroot.1 $LFS/usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/'                    $LFS/usr/share/man/man8/chroot.8
```
change to sources and delete
```
cd ..
rm -rf coreutils-9.5
```

## Diffutils-3.10
```
tar -xvf diffutils-3.10.tar.xz
cd diffutils-3.10
```

prepare, compile and install
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(./build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rf diffutils-3.10
```

## File-5.45
```
tar -xvf file-5.45.tar.xz
cd file-5.45
```

The file command on the build host needs to be the same version as the one we are building in order to create the signature file. Run the following commands to make a temporary copy of the file command:
```
mkdir build
pushd build
  ../configure --disable-bzlib      \
               --disable-libseccomp \
               --disable-xzlib      \
               --disable-zlib
  make
popd
```

prepare, compile and install
```
./configure --prefix=/usr --host=$LFS_TGT --build=$(./config.guess)

make FILE_COMPILE=$(pwd)/build/src/file

make DESTDIR=$LFS install
```

Remove the libtool archive file because it is harmful for cross compilation:
```
rm -v $LFS/usr/lib/libmagic.la
```

change to sources and delete
```
cd ..
rm -rf file-5.45
```

## Findutils-4.10.0
```
tar -xvf findutils-4.10.0.tar.xz
cd findutils-4.10.0
```

prepare, compile and install
```
./configure --prefix=/usr                   \
            --localstatedir=/var/lib/locate \
            --host=$LFS_TGT                 \
            --build=$(build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rvf findutils-4.10.0
```

## Gawk-5.3.0
```
tar -xvf gawk-5.3.0.tar.xz
cd gawk-5.3.0
```
First, ensure some unneeded files are not installed:
```
sed -i 's/extras//' Makefile.in
```
prepare, compile and install
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rvf gawk-5.3.0
```

## Grep-3.11
```
tar -xvf grep-3.11.tar.xz
cd grep-3.11
```

prepare, compile and install
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(./build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rvf grep-3.11
```

## Gzip-1.13
```
tar -xvf gzip-1.13.tar.xz
cd gzip-1.13
```

prepare, compile and install
```
./configure --prefix=/usr --host=$LFS_TGT

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rvf gzip-1.13
```

## Make-4.4.1
```
tar -xvf make-4.4.1.tar.gz
cd make-4.4.1
```

prepare, compile and install
```
./configure --prefix=/usr   \
            --without-guile \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rvf make-4.4.1
```

## Patch-2.7.6
```
tar -xvf patch-2.7.6.tar.xz
cd patch-2.7.6
```

prepare, compile and install
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)

make

make DESTDIR=$LFS install
```
change to sources and delete
```
cd ..
rm -rvf patch-2.7.6
```

## Sed-4.9
```
tar -xvf sed-4.9.tar.xz
cd sed-4.9
```

prepare, compile and install
```
./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(./build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rvf sed-4.9
```

## Tar-1.35
```
tar -xvf tar-1.35.tar.xz
cd tar-1.35
```

prepare, compile, and install
```
./configure --prefix=/usr                     \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess)

make

make DESTDIR=$LFS install
```

change to sources and delete
```
cd ..
rm -rvf tar-1.35
```

## Xz-5.6.2
```
tar -xvf xz-5.6.2.tar.xz
cd xz-5.6.2
```

prepare, compile and install
```
./configure --prefix=/usr                     \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --disable-static                  \
            --docdir=/usr/share/doc/xz-5.6.2

make

make DESTDIR=$LFS install
```

Remove the libtool archive file because it is harmful for cross compilation:
```
rm -v $LFS/usr/lib/liblzma.la
```

change to sources and delete
```
cd ..
rm -rvf xz-5.6.2
```

## Binutils-2.43.1 - Pass 2
```
tar -xvf binutils-2.43.1.tar.xz
cd binutils-2.43.1
```

Binutils building system relies on an shipped libtool copy to link against internal static libraries, but the libiberty and zlib copies shipped in the package do not use libtool. This inconsistency may cause produced binaries mistakenly linked against libraries from the host distro. Work around this issue:

```
sed '6009s/$add_dir//' -i ltmain.sh
```

Create a separate build directory again:
```
mkdir -v build
cd       build
```

Prepare, compile and install
```
../configure                   \
    --prefix=/usr              \
    --build=$(../config.guess) \
    --host=$LFS_TGT            \
    --disable-nls              \
    --enable-shared            \
    --enable-gprofng=no        \
    --disable-werror           \
    --enable-64-bit-bfd        \
    --enable-new-dtags         \
    --enable-default-hash-style=gnu

make

make DESTDIR=$LFS install
```

Remove the libtool archive files because they are harmful for cross compilation, and remove unnecessary static libraries:
```
rm -v $LFS/usr/lib/lib{bfd,ctf,ctf-nobfd,opcodes,sframe}.{a,la}
```
change to sources and delete
```
cd ../..
rm -rvf binutils-2.43.1
```

## GCC-14.2.0 - Pass 2
```
tar -xvf gcc-14.2.0.tar.xz
cd gcc-14.2.0
```

As in the first build of GCC, the GMP, MPFR, and MPC packages are required. Unpack the tarballs and move them into the required directories:

```
tar -xf ../mpfr-4.2.1.tar.xz
mv -v mpfr-4.2.1 mpfr
tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp
tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc
```

If building on x86_64, change the default directory name for 64-bit libraries to “lib”:
```
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac
```

Override the building rule of libgcc and libstdc++ headers, to allow building these libraries with POSIX threads support:
```
sed '/thread_header =/s/@.*@/gthr-posix.h/' \
    -i libgcc/Makefile.in libstdc++-v3/include/Makefile.in
```
Create a separate build directory again:
```
mkdir -v build
cd       build
```

Prepare, compile and install
```
../configure                                       \
    --build=$(../config.guess)                     \
    --host=$LFS_TGT                                \
    --target=$LFS_TGT                              \
    LDFLAGS_FOR_TARGET=-L$PWD/$LFS_TGT/libgcc      \
    --prefix=/usr                                  \
    --with-build-sysroot=$LFS                      \
    --enable-default-pie                           \
    --enable-default-ssp                           \
    --disable-nls                                  \
    --disable-multilib                             \
    --disable-libatomic                            \
    --disable-libgomp                              \
    --disable-libquadmath                          \
    --disable-libsanitizer                         \
    --disable-libssp                               \
    --disable-libvtv                               \
    --enable-languages=c,c++

make

make DESTDIR=$LFS install
```
As a finishing touch, create a utility symlink. Many programs and scripts run cc instead of gcc, which is used to keep programs generic and therefore usable on all kinds of UNIX systems where the GNU C compiler is not always installed. Running cc leaves the system administrator free to decide which C compiler to install:
```
ln -sv gcc $LFS/usr/bin/cc
```

change to sources and delete
```
cd ../..
rm -rvf gcc-14.2.0
```


## Changing Ownership
switch to root

```
sudo su
```

Currently, the whole directory hierarchy in $LFS is owned by the user lfs, a user that exists only on the host system. If the directories and files under $LFS are kept as they are, they will be owned by a user ID without a corresponding account. This is dangerous because a user account created later could get this same user ID and would own all the files under $LFS, thus exposing these files to possible malicious manipulation.

To address this issue, change the ownership of the $LFS/* directories to user root by running the following command:
```
chown --from lfs -R root:root $LFS/{usr,lib,var,etc,bin,sbin,tools}
case $(uname -m) in
  x86_64) chown --from lfs -R root:root $LFS/lib64 ;;
esac
```

## Preparing Virtual Kernel File Systems
Begin by creating the directories on which these virtual file systems will be mounted:
```
mkdir -pv $LFS/{dev,proc,sys,run}
```

Mounting and Populating /dev
```
mount -v --bind /dev $LFS/dev
```

Mounting Virtual Kernel File Systems
```
mount -vt devpts devpts -o gid=5,mode=0620 $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
```
In some host systems, /dev/shm is a symbolic link to a directory, typically /run/shm. The /run tmpfs was mounted above so in this case only a directory needs to be created with the correct permissions.

In other host systems /dev/shm is a mount point for a tmpfs. In that case the mount of /dev above will only create /dev/shm as a directory in the chroot environment. In this situation we must explicitly mount a tmpfs:

```
if [ -h $LFS/dev/shm ]; then
  install -v -d -m 1777 $LFS$(realpath /dev/shm)
else
  mount -vt tmpfs -o nosuid,nodev tmpfs $LFS/dev/shm
fi
```

## Entering the Chroot Environment
```
chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin     \
    MAKEFLAGS="-j$(nproc)"      \
    TESTSUITEFLAGS="-j$(nproc)" \
    /bin/bash --login
```

It is important that all the commands throughout the remainder of this chapter and the following chapters are run from within the chroot environment. If you leave this environment for any reason (rebooting for example), ensure that the virtual kernel filesystems are mounted as explained in Section 7.3.1, “Mounting and Populating /dev” and Section 7.3.2, “Mounting Virtual Kernel File Systems” and enter chroot again before continuing with the installation.

## Creating Directories
```
mkdir -pv /{boot,home,mnt,opt,srv}
mkdir -pv /etc/{opt,sysconfig}
mkdir -pv /lib/firmware
mkdir -pv /media/{floppy,cdrom}
mkdir -pv /usr/{,local/}{include,src}
mkdir -pv /usr/lib/locale
mkdir -pv /usr/local/{bin,lib,sbin}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -pv /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -pv /usr/{,local/}share/man/man{1..8}
mkdir -pv /var/{cache,local,log,mail,opt,spool}
mkdir -pv /var/lib/{color,misc,locate}

ln -sfv /run /var/run
ln -sfv /run/lock /var/lock

install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp
```

## Creating Essential Files and Symlinks
```
ln -sv /proc/self/mounts /etc/mtab
```

Create a basic /etc/hosts file to be referenced in some test suites, and in one of Perl's configuration files as well:
```
cat > /etc/hosts << EOF
127.0.0.1  localhost $(hostname)
::1        localhost
EOF
```

In order for user root to be able to login and for the name “root” to be recognized, there must be relevant entries in the /etc/passwd and /etc/group files.

Create the /etc/passwd file by running the following command:
```
cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/usr/bin/false
daemon:x:6:6:Daemon User:/dev/null:/usr/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/usr/bin/false
uuidd:x:80:80:UUID Generation Daemon User:/dev/null:/usr/bin/false
nobody:x:65534:65534:Unprivileged User:/dev/null:/usr/bin/false
EOF
```

Create the /etc/group file by running the following command:
```
cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
input:x:24:
mail:x:34:
kvm:x:61:
uuidd:x:80:
wheel:x:97:
users:x:999:
nogroup:x:65534:
EOF
```

Some packages need a locale.
```
localedef -i C -f UTF-8 C.UTF-8
```

Some tests in Chapter 8 need a regular user. We add this user here and delete this account at the end of that chapter.
```
echo "tester:x:101:101::/home/tester:/bin/bash" >> /etc/passwd
echo "tester:x:101:" >> /etc/group
install -o tester -d /home/tester
```

To remove the “I have no name!” prompt, start a new shell. Since the /etc/passwd and /etc/group files have been created, user name and group name resolution will now work:
```
exec /usr/bin/bash --login
```

The login, agetty, and init programs (and others) use a number of log files to record information such as who was logged into the system and when. However, these programs will not write to the log files if they do not already exist. Initialize the log files and give them proper permissions:
```
touch /var/log/{btmp,lastlog,faillog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664  /var/log/lastlog
chmod -v 600  /var/log/btmp
```

## Gettext-0.22.5
You should in / root level now(you already in chroot)
so change to sources
```
cd sources
tar -xvf gettext-0.22.5.tar.xz
cd gettext-0.22.5
```

prepare, compile and install
```
./configure --disable-shared

make

cp -v gettext-tools/src/{msgfmt,msgmerge,xgettext} /usr/bin
```
change back and delete
```
cd ..
rm -rvf gettext-0.22.5
```

## Bison-3.8.2
```
tar -xvf bison-3.8.2.tar.xz
cd bison-3.8.2
```

prepare, compile and install
```
./configure --prefix=/usr \
            --docdir=/usr/share/doc/bison-3.8.2

make

make install
```

change back and delete
```
cd ..
rm -rvf bison-3.8.2
```

## Perl-5.40.0
```
tar -xvf perl-5.40.0.tar.xz
cd perl-5.40.0
```

prepare, compile and install
```
sh Configure -des                                         \
             -D prefix=/usr                               \
             -D vendorprefix=/usr                         \
             -D useshrplib                                \
             -D privlib=/usr/lib/perl5/5.40/core_perl     \
             -D archlib=/usr/lib/perl5/5.40/core_perl     \
             -D sitelib=/usr/lib/perl5/5.40/site_perl     \
             -D sitearch=/usr/lib/perl5/5.40/site_perl    \
             -D vendorlib=/usr/lib/perl5/5.40/vendor_perl \
             -D vendorarch=/usr/lib/perl5/5.40/vendor_perl

make

make install
```

chnage back and delete
```
cd ..
rm -rvf perl-5.40.0
```

## Python-3.12.5
```
tar -xvf Python-3.12.5.tar.xz
cd Python-3.12.5
```

prepare, compile and install
```
./configure --prefix=/usr   \
            --enable-shared \
            --without-ensurepip

make

make install
```

change and delete
```
cd ..
rm -rvf Python-3.12.5
```

## Texinfo-7.1
```
tar -xvf texinfo-7.1.tar.xz
cd texinfo-7.1
```

prepare, compile and install
```
./configure --prefix=/usr

make

make install
```

change and delete
```
cd ..
rm -rvf texinfo-7.1
```

## Util-linux-2.40.2
```
tar -xvf util-linux-2.40.2.tar.xz
cd util-linux-2.40.2
```

The FHS recommends using the /var/lib/hwclock directory instead of the usual /etc directory as the location for the adjtime file. Create this directory with:

```
mkdir -pv /var/lib/hwclock
```
prepare, compile and install
```
./configure --libdir=/usr/lib     \
            --runstatedir=/run    \
            --disable-chfn-chsh   \
            --disable-login       \
            --disable-nologin     \
            --disable-su          \
            --disable-setpriv     \
            --disable-runuser     \
            --disable-pylibmount  \
            --disable-static      \
            --disable-liblastlog2 \
            --without-python      \
            ADJTIME_PATH=/var/lib/hwclock/adjtime \
            --docdir=/usr/share/doc/util-linux-2.40.2

make

make install
```
change and delete
```
cd ..
rm -rvf util-linux-2.40.2
```

# Cleaning up and Saving the Temporary System
Cleaning
```
rm -rf /usr/share/{info,man,doc}/*

find /usr/{lib,libexec} -name \*.la -delete

rm -rf /tools
```

If you have decided to make a backup, leave the chroot environment:
```
exit
```
All of the following instructions are executed by root on your host system. Take extra care about the commands you're going to run as mistakes made here can modify your host system. Be aware that the environment variable LFS is set for user lfs by default but may not be set for root.

Whenever commands are to be executed by root, make sure you have set LFS.

Before making a backup, **unmount the virtual file systems**:
```
mountpoint -q $LFS/dev/shm && umount $LFS/dev/shm
umount $LFS/dev/pts
umount $LFS/{sys,proc,run,dev}
```

Create the backup archive by running the following command:
```
cd $LFS
tar -cJpf $HOME/lfs-temp-tools-12.2.tar.xz .
```

Restore
```
cd $LFS
rm -rf ./*
tar -xpf $HOME/lfs-temp-tools-12.2.tar.xz
```
If you left the chroot environment to create a backup or restart building using a restore, remember to check that the virtual file systems are still mounted (findmnt | grep $LFS). If they are not mounted, remount them now as described in Section 7.3, “Preparing Virtual Kernel File Systems” and re-enter the chroot environment (see Section 7.4, “Entering the Chroot Environment”) before continuing.

```
findmnt | grep $LFS
mount -v --bind /dev $LFS/dev
mount -vt devpts devpts -o gid=5,mode=0620 $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

if [ -h $LFS/dev/shm ]; then
  install -v -d -m 1777 $LFS$(realpath /dev/shm)
else
  mount -vt tmpfs -o nosuid,nodev tmpfs $LFS/dev/shm
fi

chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin     \
    MAKEFLAGS="-j$(nproc)"      \
    TESTSUITEFLAGS="-j$(nproc)" \
    /bin/bash --login
```

## Man-pages-6.9.1
```
tar -xvf man-pages-6.9.1.tar.xz
cd man-pages-6.9.1
```

install man-page
```
rm -v man3/crypt*
make prefix=/usr install
```

## Iana-Etc-20240806
```
tar -xvf iana-etc-20240806.tar.gz
cd iana-etc-20240806
cp services protocols /etc
cd ..
rm -rf iana-etc-20240806
```
## Glibc-2.40
https://www.linuxfromscratch.org/lfs/view/stable/chapter08/glibc.html


## GCC-14.2.0
In this section, the test suite for GCC is considered important, but it takes a long time. First-time builders are encouraged to run the test suite. The time to run the tests can be reduced significantly by adding -jx to the make -k check command below, where x is the number of CPU cores on your system
```
su tester -c "PATH=$PATH make -k -j12 check"
```
