# Context

* URL of git repo: https://github.com/MirkoCovizzi/hello

* URL of the Launchpad PPA: https://launchpad.net/~mirkocovizzi/+archive/ubuntu/hello

* Output of the installation:

```
mirko@mirko-VMware20-1:~/Documents/projects/hello/hello$ sudo apt install hello
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  hello
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 48.4 kB of archives.
After this operation, 316 kB of additional disk space will be used.
Get:1 https://ppa.launchpadcontent.net/mirkocovizzi/hello/ubuntu noble/main arm64 hello arm64 2.10-3mirko2 [48.4 kB]
Fetched 48.4 kB in 0s (111 kB/s) 
Selecting previously unselected package hello.
(Reading database ... 166661 files and directories currently installed.)
Preparing to unpack .../hello_2.10-3mirko2_arm64.deb ...
Unpacking hello (2.10-3mirko2) ...
Setting up hello (2.10-3mirko2) ...
this is a test from Mirko Covizzi
Processing triggers for man-db (2.12.0-4build2) ...
Processing triggers for install-info (7.1-3build2) ...
mirko@mirko-VMware20-1:~/Documents/projects/hello/hello$ dpkg -S testing.sh && testing.sh
hello: /usr/bin/testing.sh
this is a test from Mirko Covizzi
mirko@mirko-VMware20-1:~/Documents/projects/hello/hello$
```

# How to prepare, patch, and publish a Debian package

## Development environment prerequisites

This guide assumes you're starting from a fresh Ubuntu Noble (24.04) system on arm64 architecture, such as:

* A virtual machine running on Apple Silicon (e.g. via VirtualBox or VMWare Fusion)

* A clean server or desktop installation with no prior packaging toolchains configured

We’ll walk through every step required to set up the system, patch an existing Debian package (`hello`), and publish it to your personal `Launchpad PPA`.

## Install prerequisites

Ensure the following packages are installed on your Ubuntu machine before starting:

```bash
sudo apt update
sudo apt install -y git dpkg-dev vim pbuilder devscripts \
                    debhelper build-essential fakeroot \
                    help2man texinfo dput
```

During installation of `pbuilder`, a prompt will appear:

```
Default mirror not found
```

Press `<enter>`.

A new window will appear, asking to add a default mirror. In the `Default mirror site:` input box, type:

```
http://ports.ubuntu.com/ubuntu-ports
```

## Prepare, build, and test the Debian package

1. Create a workspace

```bash
mkdir -p ~/Documents/projects/hello
cd ~/Documents/projects/hello
```

2. Enable source repositories

```bash
sudo vim /etc/apt/sources.list.d/ubuntu.sources
```

Change the first line:

```yaml
Types: deb
```

To:

```yaml
Types: deb deb-src
```

Then update:

```bash
sudo apt update
```

3. Fetch the package source

```bash
apt source hello
```

You will see:

```
Reading package lists... Done
NOTICE: 'hello' packaging is maintained in the 'Git' version control system at:
https://salsa.debian.org/sanvila/hello.git
Please use:
git clone https://salsa.debian.org/sanvila/hello.git
to retrieve the latest (possibly unreleased) updates to the package.
Skipping already downloaded file 'hello_2.10-3build1.dsc'
Skipping already downloaded file 'hello_2.10.orig.tar.gz'
Skipping already downloaded file 'hello_2.10.orig.tar.gz.asc'
Skipping already downloaded file 'hello_2.10-3build1.debian.tar.xz'
Need to get 0 B of source archives.
dpkg-source: info: extracting hello in hello-2.10
dpkg-source: info: unpacking hello_2.10.orig.tar.gz
dpkg-source: info: unpacking hello_2.10-3build1.debian.tar.xz
```

4. Configure `pbuilder` build output directory

Open the `pbuilder` configuration file:

```bash
sudo vim /etc/pbuilderrc
```

Add the following line at the end:

```
BUILDRESULT=..
```

This ensures that the resulting `.deb` and build artifacts will be copied to the parent directory of your source folder.
This helps keep the source tree clean and organized.

5. Create the `pbuilder` build environment

Now set up the `pbuilder` chroot for noble on arm64:

```
sudo pbuilder create --distribution noble --architecture arm64
```

This will create a clean and reproducible environment for building packages.

6. Build the source package

Navigate to the source directory and run `pbuilder`:

```bash
cd hello-2.10
sudo pbuilder build ../hello_2.10-3build1.dsc
```

This will compile the package and output several build artifacts (like `.deb`, `.changes`, `.ddeb`, and `.buildinfo`) into the parent directory.

7. Test the built package manually

Return to the parent directory:

```bash
cd ..
```

Create a temporary test directory:

```bash
mkdir test-hello
```

Extract the contents of the generated `.deb`:

```bash
dpkg-deb -x hello_2.10-3build1_arm64.deb test-hello/
```

Finally, run the `hello` binary:

```bash
./test-hello/usr/bin/hello
```

Expected output:

```
Hello, world!
```

Nice! The build worked and the binary runs correctly.

## Set up your personal Launchpad PPA

Before moving to the next section where we'll patch the `hello` package, let's first set up your PPA (Personal Package Archive) on `Launchpad`. This will allow you to publish and distribute your Debian packages so that others can easily install them via `apt`.
This section will walk you through the one-time setup steps required to publish packages from your machine to `Launchpad`, including account creation, GPG key setup, and PPA creation.

1. Create an Ubuntu One account (if you don't have one already)

Launchpad uses Ubuntu One as its authentication system.

* Visit: https://login.ubuntu.com

* Sign up for an account using your preferred email address

* Verify your email

2. Sign in to Launchpad

Once your Ubuntu One account is ready:

* Visit: https://launchpad.net

* Log in with the same Ubuntu One credentials you used in the previous step

* Launchpad will create your user profile automatically

3. Generate a GPG key (if you don't have one already)

Launchpad requires that your uploads must be signed with a GPG key.
On your machine, open a terminal and run:

```bash
gpg --full-generate-key
```

You will be shown a prompt:

* Key type: choose `RSA and RSA`

* Key size: choose `4096`

* Expiration: your choice (e.g., `1y`)

* Name/email: use the same name and email as your Ubuntu One account

* Set a secure passphrase

After generation, list your keys:

```bash
gpg --list-keys
```
Copy the fingerprint (e.g., `ABC123...`).

4. Upload your GPG key to the Ubuntu keyserver

Launchpad retrieves GPG keys from the Ubuntu keyserver.
Send the keys:

```bash
gpg --keyserver keyserver.ubuntu.com --send-keys ABC123...
```

Before moving back to Launchpad to import the key, it is necessary to wait a few minutes for the key to be available and accessible.

5. Add your GPG key to Launchpad

* Go to: https://launchpad.net/people/+me/+editpgpkeys

* In the `Fingerprint:` input box type the fingerprint

* Click `Import Key`

Launchpad will now attempt to fetch your key from the Ubuntu keyserver. Once it finds the key, it will send a confirmation email to the address associated with your key.

6. Verify your GPG key

* You will receive an email from Launchpad containing a PGP-encrypted message.

* Copy the full encrypted message into a file, for example:

```bash
cat ~/message.txt
```

Should look like:

```
-----BEGIN PGP MESSAGE-----

hQIMA6Kv...
-----END PGP MESSAGE-----
```

* Decrypt the message:

```bash
gpg --decrypt ~/message.txt
```

* At the bottom of the decrypted message, you'll find a unique verification URL, such as:

```
https://launchpad.net/token/abc123xyz
```

* Open that URL in your browser to confirm and activate your key.

7. Create a PPA

* Visit: https://launchpad.net/people/+me

* Under `Personal package archives`, click `Create a new PPA`

* Fill in:

    * PPA name: e.g., `hello`

    * Description

    * Series: leave default

    * Visibility: public

You'll now have a PPA page like:

```
https://launchpad.net/~yourusername/+archive/ubuntu/hello
```

8. Enable `arm64` builds

* Go to your PPA page, click `Change details`

* Under `Processors`, select `ARM ARMv8 (arm64)`

* Click `Save`

Success! Now the GPG key has been imported and verified, the PPA has been created and configured, and you are ready to move to the next section.

## Patching and preparing the Debian package

In a previous section, when executing `apt source hello`, you may have seen this notice:

```
Please use:
git clone https://salsa.debian.org/sanvila/hello.git
to retrieve the latest (possibly unreleased) updates to the package.
```

Let’s now clone the upstream git repository and apply our custom changes in a clean, trackable way.

1. Clone the `hello` source repository

```bash
cd ~Documents/projects/hello
git clone https://salsa.debian.org/sanvila/hello.git
cd hello
```

2. Create the `testing.sh` script

We'll add a custom script that will be included in the installed package and that can be run by users:

```bash
touch testing.sh
```

Now edit `testing.sh`:

```bash
#!/bin/bash

echo "this is a test from <your-name>" >&2
```

Make it executable:

```bash
chmod +x testing.sh
```

This script will output a test message to standard error (`STDERR`) when executed.

3. Install `testing.sh` with the package

To include the script in the installed .deb, we’ll use a file called `debian/install`.
It does not exist in the current repository, so let's create it:

```bash
touch debian/install
```

Now edit `debian/install`:

```
testing.sh usr/bin/
```

This tells the packaging tools to copy `testing.sh` into `/usr/bin/` during installation.

Then, commit the changes:

```bash
git config --global user.email "your-name@email.com"
git config --global user.name "your-name"
git checkout -b dev
git add testing.sh
git add debian/install
git commit -s -m "Add testing.sh."
```

It's important to commit the changes, because some commands in the Debian packaging workflow will refuse to run (or give warnings/errors) if the working tree is dirty.

4. Add a post-install message with `postinst`

Maintainer scripts are executable shell scripts, placed in the `debian` directory, and included inside the built `.deb` package. These are executed during install, upgrade, or removal of a package.

The `postinst` script does not exist in the current repository, so let's create it:

```bash
touch debian/postinst
```

Now edit `debian/postinst`:

```bash
#!/bin/sh
set -e

echo "this is a test from <your-name>"

#DEBHELPER#
```

Adding `set -e` at the top of the script is a best practice recommended by the Debian policy. It ensures that the script stops execution on any error, preventing further commands from running in a possibly inconsistent state.

Adding `#DEBHELPER#` at the bottom is necessary for any possible Debhelper-generated postinst logic, since Debhelper will append its code there during the build process.
Lintian is a static analysis tool for Debian packages.
Without this placeholder, you may see a Lintian warning like:

```
W: hello source: maintainer-script-lacks-debhelper-token [debian/postinst]
```

Then, commit the changes:

```bash
git add debian/postinst
git commit -s -m "Echo message in postinst."
```

## Publishing the patched Debian package

Once your changes are ready, the next steps are to version the package, build and sign the source, and upload it to your Launchpad PPA.

1. Update the changelog with `dch`

Start by incrementing the version and summarizing your changes:

```bash
dch -i
```

This will open `debian/changelog` in `vim`. Review or update:

* The release version (e.g., `2.10-3yourname1`)

* Your name and email (must match your GPG identity created in the previous sections)

* A brief changelog entry

Example:

```
hello (2.10-3yourname1) noble; urgency=medium

  * Add testing.sh.
  * Echo message in postinst.

 -- your-name <your-name@email.com>  Wed, 05 Apr 2025 13:47:00 +0000
```

Then mark it as a released version:

```bash
dch -r
```

After that, commit the changelog:

```bash
git add debian/changelog
git commit -s -m "Release as 2.10-3yourname1."
```

2. Create and describe the patch

Formalize your source modifications into a quilt patch:

```bash
dpkg-source --commit
```

When prompted, name the patch:

```
add-testing
```

Then edit the patch description. Example:

```
Add testing.sh and echo message in postinst

This patch installs a custom script and adds a confirmation message
printed to STDOUT during package installation.
```

Save and exit.

3. Sign and build the source package

List your GPG keys to get the correct key ID:

```bash
gpg --list-keys --keyid-format LONG
```

Copy the public key ID (e.g. `ABC123...`), then use it to build and sign the source package:

```bash
debuild -S -us -kABC123...
```

4. Test the build locally

If you want to verify your build using pbuilder before uploading:

```bash
sudo pbuilder build ../hello_2.10-3yourname1.dsc
```

If you install the resulting `.deb` you should see the expected behaviour.

```bash
sudo dpkg -i hello_2.10-3yourname1_arm64.deb
```

You should output like:

```
Selecting previously unselected package hello.
(Reading database ... 166661 files and directories currently installed.)
Preparing to unpack hello_2.10-3yourname1_arm64.deb ...
Unpacking hello (2.10-3mirko3) ...
Setting up hello (2.10-3mirko3) ...
this is a test from <your-name>
Processing triggers for install-info (7.1-3build2) ...
Processing triggers for man-db (2.12.0-4build2) ...
```

And by executing `testing.sh`:

```
this is a test from <your-name>
```

Then remove the package:

```bash
sudo apt remove --purge hello
```

5. Configure dput (first time only)

Create the `~/.dput.cf` file:

```bash
touch ~/.dput.cf
```

Then edit it:

```
[hello]
fqdn = ppa.launchpad.net
method = ftp
incoming = ~yourusername/ubuntu/hello/
login = anonymous
allow_unsigned_uploads = 0
```

6. Upload the source to Launchpad

```
dput hello ../hello_2.10-3yourname1_source.changes
```

You should see output like:

```
Successfully uploaded packages.
```

Awesome! This confirms that the package was accepted into Launchpad's queue.

7. Monitor build status

Visit your package listing page to track progress:

```
https://launchpad.net/~yourusername/+archive/ubuntu/hello/+packages
```

You should see the uploaded version `2.10-3yourname1` with the status `Pending`.

Afterwards, Launchpad will:

* Start CI builds for each architecture (e.g. amd64, arm64)

* Send you a confirmation email once the build begins

Once the status changes to `Published`, your package will be available for installation via `apt`.

## Testing the patched Debian package

Once your package has been successfully published to Launchpad, you can test the full installation flow using `apt`.

1. Add your Launchpad PPA

```bash
sudo add-apt-repository ppa:yourusername/hello
```

2. Update package lists

```bash
sudo apt update
```

3. Install the patched package

```bash
sudo apt install hello
```

You should see output similar to the following:

```
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  hello
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 48.4 kB of archives.
After this operation, 316 kB of additional disk space will be used.
Get:1 https://ppa.launchpadcontent.net/yourusername/hello/ubuntu noble/main arm64 hello arm64 2.10-3yourname1 [48.4 kB]
Fetched 48.4 kB in 0s (111 kB/s) 
Selecting previously unselected package hello.
(Reading database ... 166661 files and directories currently installed.)
Preparing to unpack .../hello_2.10-3yourname1_arm64.deb ...
Unpacking hello (2.10-3yourname1) ...
Setting up hello (2.10-3yourname1) ...
this is a test from <your-name>
Processing triggers for man-db (2.12.0-4build2) ...
Processing triggers for install-info (7.1-3build2) ...
```

The key confirmation is this line:

```
this is a test from <your-name>
```

This message is printed by your custom `postinst` script, confirming that the modified package installed successfully!
