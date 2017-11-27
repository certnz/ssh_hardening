# Securing SSH - the CERT NZ way

This repository contains documentation (with references) and sample configuration files which represent a more secure SSH configuration, for both SSH server and client.

CERT NZ had the following five key requirements when securing SSH:

1. Easy to use  
Do not place unnecessary burden on administrators when they are using SSH.
1.  Easy to deploy  
Do not place unnecessary burden on administrators when deploying, or when setting up new users.
1. Prevent an attacker from piggybacking off an SSH session  
In the event that an administrators workstation is compromised, an attacker should not easily be able to move laterally using SSH.
1. Protect key material from an attacker  
Prevent an attacker from being able to obtain the credentials (password / private key) for SSH authentication.
1. A way to securely copy / move private keys to other workstations  
Do not tie an administrator to one workstation because that is the only place their keys exist, but provide a more secure method than "keys on a flash drive".

With those defined requirements, CERT NZ chose to use openPGP smartcard compatible hardware tokens to protect key material, alongside 2 factor authentication. While this is an approach that CERT NZ recommends, there are other equally valid options, and it is up to you to implement the option that best protects your environment. You may have additional requirements such as cross platform compatibility. While CERT NZ has not tested other platforms, the recommendations in this repository are believed to be compatible across Linux, Mac and Windows workstations, as long as gnupg2 can be installed along with relevant drivers as necessary for your hardware token.

There is already a lot of documentation around securing SSH, and we do not seek to reproduce it here. Instead, we will explain why certain settings are important, and provide links to documentation that we at CERT NZ have found particularly useful. **Note:** Where we do provide links to external websites, CERT NZ does not endorse these individuals or companies, or the products or content on those websites, and these links are provided for reference only.

The following documentation assumes the reader is already familiar with SSH and openPGP, specifically OpenSSH and GnuPG, and has been written assuming that you are using OpenSSH 7.2 or newer, and GnuPG version 2.0.22 or newer. CERT NZ recommends using the latest version of these tools available.

If you find something that is wrong, or you feel something is missing, you can open an issue against this repository.

## Hardening your SSH server configuration
This will depend on your specific SSH server implementation and version. The main objectives are to:
* Disable connection master / multiplexing (necessary to support requirement 3)
* Remove unnecessary / older cryptographic options
* Disable options that are usually unncessary and may be exploited such as TCP forwarding and agent forwarding

For OpenSSH 7.2 and newer, the sample configuration file [config/ssh/sshd_config](config/ssh/sshd_config) may be a useful reference, but we recommend you verify each setting, rather than blindly applying it to your environment. For further reading, you may wish to check out Mozilla Security teams [SSH guide](https://wiki.mozilla.org/Security/Guidelines/OpenSSH), and Stribika handy [blog post](https://stribika.github.io/2015/01/04/secure-secure-shell.html), and documentation specific to your version of SSH.

## Hardening your SSH client configuration
In order to protect against a potentially malicious server, we apply the same care to our SSH client configuration, disabling controlmaster, older cryptography, agent forwarding and so on. See [config/ssh/ssh_config](config/ssh/ssh_config) for a sample. Again, check the documentation for your SSH client, make sure that these settings make sense and are safe in your environment.

## Two factor authentication
Two factor authentication (2FA) is a great method to achieve requirement 3, and may even be a regulatory requirement in some industries, as well as simply being best practise. By enforcing 2FA, an attacker should be unable to open an SSH session, even with access to the key material. There are many options available, and CERT NZ does not endorse any specific option. An internet search for "SSH 2 factor authentication" will yield many results.

## Using a hardware token to store your SSH keys
Using a hardware token (openPGP smartcard compatible) to store your SSH keys, rather than storing them on disk on your workstation, provides you an additional layer of protection against compromise, as well as an easy way for you to move your key material between machines more safely. Even in the event that your workstation is compromised, it should not be possible for the key material to be extracted from the hardware token. Therefore, as long as you retain physical control of the token, the key material remains safe. In this way, we address requirements 4 and 5.

CERT NZ uses Yubikey tokens from Yubico, and the following instructions only have been tested with Yubikey 4. CERT NZ do not endorse any product, nor do we offer any guarantees about whether these are fit for your purpose. You need to carry out your own evaluation and choose the product that best suits your needs, and follow the appropriate documentation for that product.

### Setting up on Linux

1. Install dependencies  
These software packages are necessary to use an openPGP smartcard with GnuPG for SSH authentication: `gnupg2` `pcscd` `scdaemon` 
You need to ensure that you have these packages installed (or compile them yourself). 
**Note:** You must use `gnupg2`, not an older version of `gnupg`. This is necessary for smartcard support. The other two packages are to enable `gnupg2` to talk to the smartcard. On some distributions, `scdaemon` may be included in the `gnupg2` package, and the packages may have different names. Using your package managers search function should find the correct packages
1. Configure gpg  
See [config/gpg/gpg.conf](config/gpg/gpg.conf) for a sample configuration file. Note that this configuration file is largely based off the [Riseup PGP recommendations](https://riseup.net/en/security/message-security/openpgp/best-practices).
1. Configure gpg-agent to enable SSH agent support  
We will be using `gpg-agent` instead of the usual `ssh-agent`. To support this, we need to configure gpg-agent to create a new socket, which will respond with SSH compatible keys. See [config/gpg/gpg-agent.conf](config/gpg/gpg-agent.conf)
1. Tell SSH to use gpg-agent rather than ssh-agent  
We need to use gpg agent, rather than other keyrings or agents such as ssh-agent or gnome-keyring. The location of this socket varies depending on what distibution you are using. You may find it at `/run/user/<uid>/gnupg/S.gpg-agent.ssh`, or `~/.gnupg/S.gpg-agent.ssh`, depending on your exact gnupg version. The method you use will vary depending on your environment. As of OpenSSH 7.3, you can set this using `IdentityAgent` config option, but for older versions, something like `export SSH_AUTH_SOCK=<path to gpg socket>` is likely to suffice. You can test this has worked successfully by checking what `env | grep SSH_AUTH_SOCK` returns.
1. Generate your key and load on to a hardware token  
You may wish to generate the key directly on the hardware token. This ensures that it cannot be leaked to the computer you are using, though it also means that you cannot backup the private key to use on another token. Alternatively, you can generate the openPGP key on a suitably secured workstation (for example, boot a laptop with no hard drive or network connectivity off a trusted liveUSB), and load it on to the hardware token. You can find sample instructions for both of these options [on Yubico website](https://developers.yubico.com/PGP/Importing_keys.html).  
**Note:** While the steps should work for you, we do disagree with some of the examples. Specifically, we recommend that you generate keys longer than 2048 bits, and we recommend you set an expiry on all openPGP keys. Also note that while the examples use `gpg`, you will need to ensure that you are using `gpg2` when working with your smartcard. Again, you may wish to read the riseup openpgp recommendations.  
If you generate your key on a workstation, note that your hardware token may have limitations about keysize or type. Ensure you generate a key that your hardware supports. For example, many smartcards are limited to RSA2048, though more modern hardware will generally support RSA4096. Support for elliptic curves such as ed25519 remains less common, though we hope this situation will improve in future.
1. Get your new SSH key  
Once you have your openPGP key loaded on your hardware token, you should be able to get an SSH compatible key using the command `gpg2 --export-ssh-key <id>`
If your environment has been set up correctly, you should see the same key using the command `ssh-add -L`
At this point, you should be able to copy the new public key to the computers that you need to authenticate against, and use this key to authenticate.
1. Change the PIN  
Make sure you have set a strong PIN or passphrase to protect your key. You can change it using the `gpg2 --card-edit` command, and the `passwd` subcommand.
1. Enforce touch to sign  
One nice feature on the Yubikey is the ability to enforce "touch to sign". This provides a simple "proof of presence" check, to mitigate the possiblity of a remote attacker using your PGP key to open SSH connections. **Note:** touch to sign functionality is only a partial mitigation, and not a full replacement for 2 factor authentication. If you wish to use this functionality, Yubico provide a shell script, details of which can be found on [Yubico's Card Edit Page](https://developers.yubico.com/PGP/Card_edit.html) in the "Yubikey 4 touch" section

## References

A list of the external links, in one place for good measure.

* [Mozilla OpenSSH recommendations](https://wiki.mozilla.org/Security/Guidelines/OpenSSH)
* [Stribika SSH blog entry](https://stribika.github.io/2015/01/04/secure-secure-shell.html)
* [Riseup openPGP recommendations](https://riseup.net/en/security/message-security/openpgp/best-practices)
* [Arch Linux wiki GnuPG page](https://wiki.archlinux.org/index.php/GnuPG)
* [Yubico openPGP documentation](https://developers.yubico.com/PGP/)
