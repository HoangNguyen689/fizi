+++
date = '2025-04-16T00:23:00+09:00'
draft = false
title = 'Fixing UID Mapping Issues in Lima for Rootless Docker'
tags = ['docker', 'lima', 'rootless', 'uidmap']
+++

## The Symptom

You’ve installed Lima using the default [Docker template](https://github.com/lima-vm/lima/blob/4c82f35d50decbc2fc07af881317e687f7161344/templates/docker.yaml#L1), and when launching your instance, it gets stuck here:

```
INFO[0057] [hostagent] Waiting for the optional requirement 1 of 1: "user probe 1/1"
```

That cryptic line basically means: Docker’s not ready yet.

Let’s SSH into the instance to investigate:

```
limactl shell local
```

Then try to manually install Docker in rootless mode:

```
dockerd-rootless-setuptool.sh install
```

Oops:

```
[rootlesskit:parent] error: failed to setup UID/GID map: newuidmap 3326 [0 222586300 1 1 524288 1073741824] failed: newuidmap: write to uid_map failed: Invalid argument
: exit status 1
[ERROR] RootlessKit failed, see the error messages and https://rootlesscontaine.rs/getting-started/common/ . Set --force to ignore
```

So what happened here?

## The Problem: UID Mapping Failure

Docker in rootless mode relies on [User Namespaces](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) to emulate root privileges in a safe way. Essentially, inside the namespace, a regular user appears to have uid 0 (i.e., root), but outside that sandbox, they’re just a regular user with a mapped ID.

This mapping is defined in /etc/subuid and /etc/subgid, and if those mappings are misconfigured or out of bounds, things break.

In our case, the UID mapping tried to use this:

```
cat /etc/subuid

user:524288:1073741824
```

Looks fine, right? But now check your actual UID:

```
id -u

222586300
```

That’s over 200 million—quite high. So is this mapping range even valid? Let’s look at /etc/login.defs:

```
grep -E 'UID_MIN|UID_MAX' /etc/login.defs

UID_MIN                  1000
UID_MAX                 60000
#SYS_UID_MIN              100
#SYS_UID_MAX              999
SUB_UID_MIN                100000
SUB_UID_MAX             600100000
```

Aha! The max value for subordinate UIDs is about 600 million. Our mapping range (524,288 → +1 billion) exceeds this limit. That’s why newuidmap fails with an “Invalid argument”.

## The Fix: Add Proper UID/GID Ranges

To solve this, you can explicitly set a valid UID mapping that fits within the allowed range. Add the following lines after the *systemctl --user start dbus* line in your Lima config:

```
echo "$USER:100000:65536" | sudo tee /etc/subuid
echo "$USER:100000:65536" | sudo tee /etc/subgid
```

This gives the user a proper UID/GID mapping (from 100000 to 165536), which works for most rootless Docker setups.

Restart your Lima instance, and things should now work.

## Final Thoughts

Rootless Docker is awesome for security, but it comes with quirks—especially around UID mapping. The key takeaway here is: Don’t trust the defaults blindly. Always check if your user ID and range are within bounds.

Also, wouldn't it be nice if newuidmap just said “UID out of range” instead of "Invalid argument"? 😅


Btw:
Can you guess the number 524288 and 1073741824?
Hzzz, they are 2^19 and 2^30. ^^
