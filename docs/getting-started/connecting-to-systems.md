# Connecting to Clusters

## Quick Start

The usual workflow is to use SSH to connect to Casper or Derecho from a terminal in your local workstation or laptop.

```
 $ ssh -l <user-name> casper.hpc.ucar.edu
```
or
```
 $ ssh <user-name>@casper.hpc.ucar.edu
```
if the username on the local workstation is the same as on the NCAR systems, then that can be omitted. Similarly, to log on to Derecho, issue commands like this
```
 $ ssh <user-name>@derecho.hpc.edu
```
`ssh` will prompt you for a password and ask you to confirm using the two-factor authentication with DUO as described in [Authenticating with Duo](accounts/duo/index.md#hpc-and-ssh-logins). For various ways of streamlining the connection, read the rest of this section.

## Setting up local SSH configuration

On Linux and Mac systems, you can add features to your `ssh` connection or shorten your command line by adding to your workstation's `~/.ssh/config` file. For example, this
```
host casper
     HostName casper.hpc.ucar.edu

host derecho
     HostName derecho.hpc.ucar.edu
```
will allow you to connect by using the short-hand `ssh <user-name>@derecho`. As suggested in an NCAR HPC User Group blog post, "Streamling two-factor authentication with `ssh`", you can limit the number of authentications for each cluster separately by including this in your `~/.ssh/config` file.
```
host *
    ControlMaster auto
    ControlPath ~/.ssh/controlpath-%r@%h:%p
    ControlPersist 12h
    ServerAliveInterval 120s
    ServerAliveCountMax 5
```
This creates a unique ControlPath file in `~/.ssh/` for each cluster. With this enabled, you will only be required to authenticate on the first connection to each cluster, `casper` or `derecho` from your local computer. If you are connecting to both clusters, you will need to authenticate twice for multiple connections to both. Using `bao-getkey` on a Mac, as described below, it is possible limit that to one authentication for connecting multiple times to both clusters.

## OpenBao Certificates

`bao-getkey` uses OpenBao's HTTP API to request a certificate-signed SSH key for passwordless SSH within the systems managed by the High-Performance Systems Group (HSG) at NSF NCAR. This includes the Derecho and Casper Clusters.

Specifically, this script does the following, all through our OpenBao instance:
1. Authenticate the user with NCAR's central authentication system
2. Request a new SSH key that is signed with a trusted certificate
3. Add the new private key and its certificate to your running `ssh-agent`

It is a BASH script installed in the default PATH on Casper and Derecho. You may either copy it to your local Linux or Mac computer and put in a directory in your PATH to run it locally, or you can run it from the cluster and have the key forwarded back to your local computers agent. These procedures will be described below.

### Usage
```
    bao-getkey [-a] [-u <username>]
```
Options:

*  `-a` - Request a certificate with necessary options for administrators. This will only work for system administrators
*  `-u <username>` - Specify the username to authenticate with. This is useful if your username on the NSF NCAR clusters is different than your on your local system. Most users who are not NCAR employees will need to use this option.

### Running bao-getkey from Derecho or Casper

Since `bao-getkey` is installed on the Derecho and Casper login nodes and is already in your `PATH` there. This is the recommended route for anyone who can't conveniently run the script on their own machine, Windows users in particular.

The key goes into whichever `ssh-agent` the `SSH_AUTH_SOCK` environment variable points at. If you simply log in to a login node and run `bao-getkey`, the key ends up in an agent on that login node, which does nothing to help you log in from your own machine. To get the key into the agent on your own machine, connect with agent forwarding enabled using `ssh -A`, and `ssh-add` then adds the key back through the forwarded connection.

First make sure you have an agent running that can accept the key from `bao-getkey`. On local systems with BASH or ZSH, you simply run `eval $(ssh-agent)` first. For terminals with TCSH shells, the corresponding command is ``eval `ssh-agent -c` ``. On Windows, make sure the OpenSSH agent is running first. In PowerShell, as Administrator:
```
    Get-Service ssh-agent | Set-Service -StartupType Automatic
    Start-Service ssh-agent
```
Then connect to the cluster with agent forwarding and run the script. You will need to authenticate normally for this first login:
```
    $ ssh -A <username>@derecho.hpc.ucar.edu
    $ bao-getkey
    $ exit
```
Note that you don't need the `-u` option here, since the script defaults to your username on the cluster, which is the one you want. In this case you will need to authenticate twice: once for the `ssh` and once for `bao-getkey`. Back on your own machine, `ssh-add -L` will show the new certificate, and logins will not prompt for a password until it expires. This workflow relies on your local agent accepting keys added over the forwarded connection. The OpenSSH client and agent included with Windows support this. PuTTY and Pageant may not, so PuTTY users may need to use the OpenSSH client instead.

If your local computer has `$SSH_AUTH_SOCK` set when your user environment is initialized, then the agent is available from any terminal or shell. You can check this with `echo $SSH_AUTH_SOCK`. If this is set to a system directory and is available from the first `ssh -A` connection that invoked `bao-getkey`, then the OpenBao key and certificate are available from any terminal.
