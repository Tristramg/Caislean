# Security-related design choices in Caisleán

Among the configuration files provided by Caisleán for its various included
pieces of software, choices were made with security as a high criterion. This
section provides more details about changes that were made with security in
mind:

- reducing risks of illegitimate access on the server running Caisleán;
- securing the content of communcations between the server and its users;
- having the administrator to perform some important tasks by hand to ensure a
  sufficient awareness about the important files stored on the local system.

## Security of the system running Caisleán

### Automatic update notifications

Using `apticron`, available packages upgrades are periodically checked, and an
e-mail notification is sent to the administrator whenever packages can be
upgraded.

Software upgrades often fix security issues, making it important to remind the
administrator to perform them. They are one of the most basic measures for
keeping a system reasonable secure.

Since upgrades are often coupled with necessary manual operations (rebooting the
system, restarting some services), the action of upgrading packages is left to
the administrator.

This is set up in the role `common`.

### Rootkit and filesystem alteration checking

Using `rkhunter` and `chkrootkit`, periodic checks are made in order to spot
illegitimate changes in the filesystem or apparition of well-known pieces of
malware.

Both of these tools are used because their detection mechanisms differ.
`rkhunter` checks for any suspicious changes in files on the system, regardless
of the content of the changes. For instance, an alert will be raised if a binary
under `/usr/bin` is changed outside of a system upgrade. On the other hand,
`chkrootkit` maintains a list of known malware file hashes and behavior and
raises an alert in case of a match.

Alerts are sent by e-mail to the administrator. This is set up in the role
`common`.

### SSH access hardening

The SSH server configuration is changed so that:

- no password-based authentication is permitted, to remove the risk of
  bruteforce attacks;
- `root` login is forbidden, so that a SSH key compromission does not lead to
  direct compromission of the `root` account;
- only one login is allowed: the one with which Ansible connects to the system,
  to make sure other logins cannot be used to access the system.

This is set up by the role `common`.

### Firewall

Using `ufw` and `iptables`, most of the trafic to and from the server is
blocked. TCP and UDP ports are by default blocked inbound and outbound, excepted
22 and 25 inbound as well as 80 and 443 outbound, as HTTP access is required for
system upgrades.

These measures may notably prevent illegitimate software from from generatic
network trafic, in case they are remotely controled.

This is set up by the role `common`, and more ports are allowed in roles setting
up network services, such as OpenVPN.

### PHP hardening

Caisleán includes several web-based services that require PHP: Roundcube,
Wordpress, Owncloud and others. The web server is Nginx and PHP pages are
processed by PHP-FPM.

As PHP applications may be the source of security issues, it is a good idea to
try to mitigate in advance potiential illegitimate access to PHP functions. For
this, each PHP-based application in Caisleán is associated with a unique PHP-FPM
process running with its own UNIX user and a separate session directory. In
addition, each have a white list of directories it is allowed to access to, in
order to reduce illegitimate access to files. These measures are set by each
role associated to a particular service, such as `roundcube`, `owncloud`, etc.

To reduce the risks related to insecure PHP applications, the most potentially
dangerous PHP functions are forbidden by the PHP configuration. This is set by
the role `php-fpm`.

### Other system hardening

The role `common` also brings the following improvements:

- installation of the _backports_ repository for latest software updates;
- default file creation mask of _077_ (no access authorized to anyone not owning
  the file) through PAM's `umask` module;
- hardening of several `sysctl` parameters;
- removal of several unwanted packages, including `exim4`, `rpcbind` and
  `nfs-common`.

## Security of communications between the server and its users

### TLS cipher list

Helping securing exchanges between the server and its users is done through a
careful selection of available TLS ciphers. The cipherlist first privileges
elliptic curve Diffie-Hellmann (ECDH) based ciphers sorted by decreasing key
size order, followed by "classical" Diffie-Hellman (DH) based ciphers. Ciphers
using SHA1
[HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) are
however placed at the end of the list, as this algorithm's security is
increasingly questioned.

A 3DES cipher is appended for HTTPS, to ensure compatibility with Internet
Explorer 8, despite its questionable security. It is the only one that does not
provide forward secrecy.

The cipherlists are provided in the configuration files for each service, in the
individual associated roles.

## Manual administration tasks for better awareness

Some tasks were deliberately left to the system administrator, in order to make
sure they are conscious of the place where sensitive files are stored on the
local system.

### TLS private keys and certificates

The TLS setup used by Caisleán includes:

- a certification authority (CA) certificate along with its private key;
- a certificate for the server, signed by this CA, along with its private key;
- a file containing Diffie-Hellmann parameters;
- optionally, other certificates signed by the CA, notably in case of VPN
  clients being allow through the use of TLS certificates.

Among these, the private keys are particularly sensitive and must be protected
on the local machine used by the server administrator. For this reason, their
creation is left to the administrator, to make sure that their location is
clearly known and that a passphrase is provided. The `tls` role documentation
provides the necessary commands to create and securely store theses files, and
the `tls/` directory at the root of the repository can be used as empty TLS CA
tree, as it notably contains a pre-configured `openssl.cnf` file.

The CA public certificate, the server certificate and private key and the
Diffie-Hellmann parameters are pushed to the server by the `tls` role. These
files are subsequently needed by all TLS-enabled services (Nginx, Postfix,
Prosody, etc.).

### DKIM private key

[DKIM](https://en.wikipedia.org/wiki/DKIM) e-mail certification is used to
certify that a message was legitimately sent from a given domain. It relies on
adding a header to every sent e-mail, containing a cryptographic signature based
on a private key. The corresponding public key is available as a specific DNS
entry for the given domain, and is read by the receiving server to check the
signature.

Similarly to the TLS keys, it is asked to the administrator to generate the DKIM
key pair on the local system. The documentation for the role `virtualmail`
provides OpenSSL-based commands to generate these keys, and the role itself
pushes them to the administered server.

### PGP backup private key and backup server security

Caisleán provides the possibility to make encrypted incremental backups thanks
to Backupninja and Duplicity. Backupninja is the backup scheduler, and the
backup itself is made by Duplicity, which requires a PGP key pair.

The PGP private and public keys used for backup must be stored as files on the
local system, and are subsequently uploaded to the server. This upload as well
as the documentation on how to export the PGP keys to files are part of the
`backupninja` role. Its documentation also includes instructions on setting up
the backup server's SSH-based access to allow the backups to be pushed to it.

# What we talk about when we talk about security

Caisleán helps set up in few simple steps one or more secure servers. This means
that the cookbook includes a whole range of **best practices for basic
security**, with tweakings regarding TLS cipher lists, web server security
options, files and directories permissions and ownership, etc.

Nevertheless, depending on your and your users' threat model, it is worth
pointing out that basic server security is **just one part** of the setup that
is needed for attaining a suitable security level for yourself and your users.

Therefore, we strongly recommend you to **inform your users** on how to improve
their security through end-to-end encryption (for example advising them to read
manuals on
[GPG](https://learn.equalit.ie/wiki/To_send_an_email_that_no_one_but_me_and_the_recipient_can_read)
and
[OTR](https://learn.equalit.ie/wiki/I_want_to_know_about_options_for_private_chat)),
as well as to warn them about the limits of personal VPNs (you can find a good
explanation [here](https://help.riseup.net/it/vpn), in the section "Limitations
to using RiseupVPN").

You should also understand that having implemented basic security settings in
the server you run does not protect you from every possible risk. Depending on
the **political context** where you and your community live and where your
activities take place, as well as where your server is located, your server
might be seized or intercepted by authorities and there might be legal
proceedings against you, your organization and/or your users.

Before you set up your server and start offering services to your community, it
is therefore strongly advisable to **consider the risks** that you may be facing
and to make decisions accordingly as regards the location of your servers, the
services you want to offer (which may be for instance illegal in some states),
the activities you want to take place on your server and also your legal status
(in some countries you could be considered a provider and further rules could
apply to your case).

Finally consider that a strong user community, if well informed, may be one of
your main strengths in many critical situations.