![Docker automated](https://img.shields.io/docker/automated/netresearch/sftp.svg)![Docker build](https://img.shields.io/docker/build/netresearch/sftp.svg)![Docker stars](https://img.shields.io/docker/stars/netresearch/sftp.svg)![Docker pulls](https://img.shields.io/docker/pulls/netresearch/sftp.svg)


# Netresearch SFTP
This repository provides an SFTP ([SSH file Transfer Protocol](https://en.wikipedia.org/wiki/SSH_File_Transfer_Protocol)) server with [OpenSSH](https://en.wikipedia.org/wiki/OpenSSH).

## Installation & Configuration
- see section [Examples](#examples)

## Usage
- Required: define users as command arguments, STDIN or mounted in `/etc/sftp/users.conf` (syntax: `user:pass[:e][:uid[:gid[:dir1[,dir2]...]]]...`).
    - Set UID/GID manually for your users if you want them to make changes to your mounted volumes with permissions matching your host filesystem.
    - Add directory names at the end, if you want to create them under the user's home directory. Perfect when you just want a fast way to upload something.
- Optional (but recommended): mount volumes.
    - The users are chrooted to their home directory, so you can mount the volumes in separate directories inside the user's home directory (/home/user/**mounted-directory**) or just mount the whole **/home** directory. Just remember that the users can't create new files directly under their own home directory, so make sure there are at least one subdirectory if you want them to upload files.
    - For consistent server fingerprint, mount your own host keys (i.e. `/etc/ssh/ssh_host_*`)

## Examples
### Simplest docker run example
    docker run -p 22:22 -d netresearch/sftp foo:pass:::upload
User "foo" with password "pass" can login with sftp and upload files to a folder called "upload". No mounted directories or custom UID/GID. Later you can inspect the files and use `--volumes-from` to mount them somewhere else (or see next example).

### Sharing a directory from your computer
Let's mount a directory and set UID (we will also provide our own hostkeys):

    docker run \
        -v /host/upload:/home/foo/upload \
        -v /host/ssh_host_rsa_key:/etc/ssh/ssh_host_rsa_key \
        -v /host/ssh_host_rsa_key.pub:/etc/ss/ssh_host_rsa_key.pub \
        -p 2222:22 -d netresearch/sftp \
        foo:pass:1001

#### Using Docker Compose:

    sftp:
        image: netresearch/sftp
        volumes:
            - /host/upload:/home/foo/upload
            - /host/ssh_host_rsa_key:/etc/ssh/ssh_host_rsa_key
            - /host/ssh_host_rsa_key.pub:/etc/ssh/ssh_host_rsa_key.pub
        ports:
            - "2222:22"
        command: foo:pass:1001

#### Logging in

The OpenSSH server runs by default on port 22, and in this example, we are
forwarding the container's port 22 to the host's port 2222. To log in with the
OpenSSH client, run: `sftp -P 2222 foo@<host-ip>`

### Store users in config

    docker run \
        -v /host/users.conf:/etc/sftp/users.conf:ro \
        -v mySftpVolume:/home \
        -v /host/ssh_host_rsa_key:/etc/ssh/ssh_host_rsa_key \
        -v /host/ssh_host_rsa_key.pub:/etc/ssh/ssh_host_rsa_key.pub \
        -p 2222:22 -d netresearch/sftp

/host/users.conf:

    foo:123:1001:100
    bar:abc:1002:100
    baz:xyz:1003:100

### Encrypted password
Add `:e` behind password to mark it as encrypted. Use single quotes if using terminal.

    docker run \
        -v /host/share:/home/foo/share \
        -p 2222:22 -d netresearch/sftp \
        'foo:$1$0G2g0GSt$ewU0t6GXG15.0hWoOX8X9.:e:1001'

Tip: you can use [atmoz/makepasswd](https://hub.docker.com/r/atmoz/makepasswd/) to generate encrypted passwords:
`echo -n "your-password" | docker run -i --rm atmoz/makepasswd --crypt-md5 --clearfrom=-`

### Logging in with SSH keys
Mount public keys in the user's `.ssh/keys/` directory. All keys are
automatically appended to `.ssh/authorized_keys` (you can't mount this file
directly, because OpenSSH requires limited file permissions). In this example,
we do not provide any password, so the user `foo` can only login with his SSH
key.

    docker run \
        -v /host/id_rsa.pub:/home/foo/.ssh/keys/id_rsa.pub:ro \
        -v /host/id_other.pub:/home/foo/.ssh/keys/id_other.pub:ro \
        -v /host/share:/home/foo/share \
        -p 2222:22 -d netresearch/sftp \
        foo::1001

### Providing your own SSH host key
This container will generate new SSH host keys at first run. To avoid that your
users get a MITM warning when you recreate your container (and the host keys
changes), you can mount your own host keys.

    docker run \
        -v /host/ssh_host_ed25519_key:/etc/ssh/ssh_host_ed25519_key \
        -v /host/ssh_host_rsa_key:/etc/ssh/ssh_host_rsa_key \
        -v /host/share:/home/foo/share \
        -p 2222:22 -d netresearch/sftp \
        foo::1001

Tip: you can generate your keys with these commands:

    ssh-keygen -t ed25519 -f /host/ssh_host_ed25519_key < /dev/null
    ssh-keygen -t rsa -b 4096 -f /etc/ssh/ssh_host_rsa_key < /dev/null


### Execute custom scripts or applications
Put your programs in `/etc/sftp.d/` and it will automatically run when the container starts.
See next section for an example.

### Bindmount dirs from another location
If you are using `--volumes-from` or just want to make a custom directory
available in user's home directory, you can add a script to `/etc/sftp.d/` that
bindmounts after container starts.

    #!/bin/bash
    # File mounted as: /etc/sftp.d/bindmount.sh
    # Just an example (make your own)

    function bindmount() {
        if [ -d "$1" ]; then
            mkdir -p "$2"
        fi
        mount --bind $3 "$1" "$2"
    }

    # Remember permissions, you may have to fix them:
    # chown -R :users /data/common

    bindmount /data/admin-tools /home/admin/tools
    bindmount /data/common /home/dave/common
    bindmount /data/common /home/peter/common
    bindmount /data/docs /home/peter/docs --read-only


***We thank atmoz for providing this great docker container!***


