## Checking host system requeriments

First get into root mode

    sudo su -

Now copy and paste the following script in the terminal

    cat > version-check.sh << "EOF"
    #!/bin/bash
    # A script to list version numbers of critical development tools
    
    # If you have tools installed in other directories, adjust PATH here AND
    # in ~lfs/.bashrc (section 4.4) as well.
    
    LC_ALL=C
    PATH=/usr/bin:/bin
    
    bail() { echo "FATAL: $1"; exit 1; }
    grep --version > /dev/null 2> /dev/null || bail "grep does not work"
    sed '' /dev/null || bail "sed does not work"
    sort /dev/null || bail "sort does not work"
    
    ver_check()
    {
      if ! type -p $2 &>/dev/null
      then
        echo "ERROR: Cannot find $2 ($1)"; return 1;
      fi
      v=$($2 --version 2>&1 | grep -E -o '[0-9]+\.[0-9\.]+[a-z]*' | head -n1)
      if printf '%s\n' $3 $v | sort --version-sort --check &>/dev/null
      then
        printf "OK: %-9s %-6s >= $3\n" "$1" "$v"; return 0;
      else
        printf "ERROR: %-9s is TOO OLD ($3 or later required)\n" "$1";
        return 1;
      fi
    }
    
    ver_kernel()
    {
      kver=$(uname -r | grep -E -o '^[0-9\.]+')
      if printf '%s\n' $1 $kver | sort --version-sort --check &>/dev/null
      then
        printf "OK: Linux Kernel $kver >= $1\n"; return 0;
      else
        printf "ERROR: Linux Kernel ($kver) is TOO OLD ($1 or later required)\n" "$kver";
        return 1;
      fi
    }
    
    # Coreutils first because --version-sort needs Coreutils >= 7.0
    ver_check Coreutils sort 8.1 || bail "Coreutils too old, stop"
    ver_check Bash bash 3.2
    ver_check Binutils ld 2.13.1
    ver_check Bison bison 2.7
    ver_check Diffutils diff 2.8.1
    ver_check Findutils find 4.2.31
    ver_check Gawk gawk 4.0.1
    ver_check GCC gcc 5.2
    ver_check "GCC (C++)" g++ 5.2
    ver_check Grep grep 2.5.1a
    ver_check Gzip gzip 1.3.12
    ver_check M4 m4 1.4.10
    ver_check Make make 4.0
    ver_check Patch patch 2.5.4
    ver_check Perl perl 5.8.8
    ver_check Python python3 3.4
    ver_check Sed sed 4.1.5
    ver_check Tar tar 1.22
    ver_check Texinfo texi2any 5.0
    ver_check Xz xz 5.0.0
    ver_kernel 4.19
    
    if mount | grep -q 'devpts on /dev/pts' && [ -e /dev/ptmx ]
    then echo "OK: Linux Kernel supports UNIX 98 PTY";
    else echo "ERROR: Linux Kernel does NOT support UNIX 98 PTY"; fi
    
    alias_check() {
      if $1 --version 2>&1 | grep -qi $2
      then printf "OK: %-4s is $2\n" "$1";
      else printf "ERROR: %-4s is NOT $2\n" "$1"; fi
    }
    echo "Aliases:"
    alias_check awk GNU
    alias_check yacc Bison
    alias_check sh Bash
    
    echo "Compiler check:"
    if printf "int main(){}" | g++ -x c++ -
    then echo "OK: g++ works";
    else echo "ERROR: g++ does NOT work"; fi
    rm -f a.out
    
    if [ "$(nproc)" = "" ]; then
      echo "ERROR: nproc is not available or it produces empty output"
    else
      echo "OK: nproc reports $(nproc) logical cores are available"
    fi
    EOF
    bash version-check.sh

## Creating the partitions

3 partitions, same shi as artix

To create a file system on the partition type the following command for every partition you created;

    mkfs -v -t ext4 /dev/sdx

Now for mounting the partitions, create the mount point with these commands;

    mount -v -t ext4 /dev/sdx2 /mnt
    mkdir /mnt/home
    mkdir /mnt/boot
    mount -v -t ext4 /dev/sdx1 /mnt/boot
    mount -v -t ext3 /dev/sdx3 /mnt/home

# Installing packages

To create this directory, execute the following command, as user `root`, before starting the download session;

    mkdir -v /mnt/sources
Make this directory writable and sticky. “Sticky” means that even if multiple users have write permission on a directory, only the owner of a file can delete the file within a sticky directory. The following command will enable the write and sticky modes:

    chmod -v a+wt /mnt/sources

To download all of the packages and patches by using [wget-list](https://www.linuxfromscratch.org/lfs/view/development/wget-list) as an input to the wget command, use: 

    wget --input-file=wget-list --continue --directory-prefix=/mnt/sources

Additionally, starting with `LFS-7.0`, there is a separate file, [md5sums](http://www.linuxfromscratch.org/lfs/view/stable/md5sums), which can be used to verify that all the correct packages are available before proceeding. Place that file in $LFS/sources and run:

    pushd /mnt/sources
      md5sum -c md5sums
    popd

- For some reason some packages get failed (maybe removing this step in the future)

## Creating a limited directory layout in the LFS Filesystem

Create the required directory layout by issuing the following commands as `root`;

    mkdir -pv /mnt/{etc,var} /mnt/usr/{bin,lib,sbin}

    for i in bin lib sbin; do
      ln -sv usr/$i /mnt/$i
    done

    case $(uname -m) in
      x86_64) mkdir -pv /mnt/lib64 ;;
    esac

Also;

    mkdir -pv /mnt/usr/lib{,x}32
    ln -sv usr/lib32 /mnt/lib32
    ln -sv usr/libx32 /mnt/libx32

Still acting as `root`, create that directory with this command:

    mkdir -pv /mnt/tools

## Adding a user to LFS

When logged in as `root`, making a single mistake can damage or destroy a system. Therefore, the packages in the next two chapters are built as an unprivileged user. You could use your own user name. We will create a new user called `user01` as a member of a new group (named `lfs`) and run commands as `lfs` during the installation process. As `root`, issue the following commands to add the new user:

    groupadd lfs
    useradd -s /bin/bash -g lfs -m -k /dev/null user01



### This is what the command line options mean:

- `s /bin/bash`

This makes bash the default shell for user lfs.

- `g lfs`

This option adds user lfs to group lfs.

- `m`

This creates a home directory for lfs.

- `k /dev/null`

This parameter prevents possible copying of files from a skeleton directory (the default is /etc/skel) by changing the input location to the special null device.

- `user01`

This is the name of the new user.

To set a password for `user01` issue the following command as the `root` to set the password;

    passwd user01

Grant `user01` full access to all the directories under `/mnt` by making lfs the owner;

    chown -v user01 /mnt/{usr{,/*},lib,var,etc,bin,sbin,tools}
    case $(uname -m) in
      x86_64) chown -v user01 /mnt/lib64 ;;
    esac

Next, start a shell running as `lfs user`. This can be done by logging in as `user01` on a virtual console, or with the following substitute/switch user command:

    su - user01

## Setting up the enviroment

Set up a good working environment by creating two new startup files for the bash shell. While logged in as `lfs user`, issue the following command to create a new `.bash_profile`:

    cat > ~/.bash_profile << "EOF"
    exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
    EOF

The new instance of the shell is a non-login shell, which does not read, and execute, the contents of the `/etc/profil`e or `.bash_profile` files, but rather reads, and executes, the `.bashrc` file instead. Create the `.bashrc` file now: 

    cat > ~/.bashrc << "EOF"
    set +h
    umask 022
    LC_ALL=POSIX
    LFS_TGT=x86_64-lfs-linux-gnu
    LFS_TGT32=i686-lfs-linux-gnu
    LFS_TGTX32=x86_64-lfs-linux-gnux32
    PATH=/usr/bin
    if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
    PATH=/mnt/tools/bin:$PATH
    CONFIG_SITE=/mnt/usr/share/config.site
    export LC_ALL LFS_TGT PATH
    EOF

Then enter:

    [ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE

### About SBUs

For compiling in `LFS` (or any large source build), the general rule of thumb is to use 2x the number of logical cores for maximum parallelism. So, if you have 8 logical cores, you would typically set the number of jobs to 16 (i.e., `-j16`).

    export MAKEFLAGS='-jx'

or by building with:

    make -jx

### Binutils

    cd /mnt/sources/
    tar -xvf binutils-2.43.1.tar.xz
    cd binutils-2.43.1/
    mkdir -v build
    cd build/

And copy paste the following script;

    ../configure --prefix=/mnt/tools \
    --with-sysroot=/mnt \
    --target=$LFS_TGT \
    --disable-nls \
    --enable-gprofng=no \
    --disable-werror
    
Continue with compiling the package;

    time make

Install the package;

    make install

And you can remove the directory

    rm -fr binutils-2.43.1/

## GCC

    tar -xvf gcc-14.2.0.tar.xz
    cd gcc-14.2.0/

run one by one;

    tar -xf ../mpfr-4.2.0.tar.xz
    mv -v mpfr-4.2.0 mpfr
    tar -xf ../gmp-6.3.0.tar.xz
    mv -v gmp-6.3.0 gmp
    tar -xf ../mpc-1.3.1.tar.gz
    mv -v mpc-1.3.1 mpc

On x86_64 hosts, set the default directory name for `64-bit` libraries to “`lib`”;

    case $(uname -m) in
        x86_64)
            sed -e '/m64=/s/lib64/lib/' \
            -i.orig gcc/config/i386/t-linux64
        ;;
    esac

The GCC documentation recommends building GCC in a dedicated build directory;

    mkdir -v build
    cd build

Prepare GCC for compilation:

    ../configure \
    --target=$LFS_TGT \
    --prefix=/mnt/tools \
    --with-glibc-version=2.38 \
    --with-sysroot=/mnt \
    --with-newlib \
    --without-headers \
    --enable-default-pie \
    --enable-default-ssp \
    --disable-nls \
    --disable-shared \
    --disable-multilib \
    --disable-threads \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libquadmath \
    --disable-libssp \
    --disable-libvtv \
    --disable-libstdcxx \
    --enable-languages=c,c++

Compile GCC by running:

    time make

Install the package:

    make install
