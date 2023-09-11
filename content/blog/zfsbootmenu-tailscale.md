+++
title = "Remote Access to ZFSBootMenu via Tailscale"
date = "2023-09-11"
+++

When booting computers with ZFS as the root filesystem, I like to use [ZFSBootMenu](https://zfsbootmenu.org) as bootloader.
Because it runs in a Linux ramdisk, it is [possible to set up networking and an SSH server](https://docs.zfsbootmenu.org/en/v2.2.x/guides/general/remote-access.html) and connect remotely to, for example, allow unlocking of an encrypted ZFS pool.
This is great when I have access to the local network in question (either directly or via a jump host), but I don't always have that kind of access.
To ensure I always have access, I set about integrating ZFSBootMenu's remote access capability with my [Tailnet](https://tailscale.com).

<!-- more -->

My ZFSBootMenu images are already [generated with mkinitcpio](https://docs.zfsbootmenu.org/en/v2.2.x/guides/general/mkinitcpio.html), so I created a simple, modular [mkinitcpio hook](https://github.com/classabbyamp/mkinitcpio-tailscale) that connects to Tailscale.
Tailscale makes it very hard to do this in the most secure way (ephemeral auth key), as auth keys and API keys (to create more auth keys) are capped at 90 days expiration.
As a compromise, I chose to use Tailscale's ACL tags to ensure the Tailscale state and private keys are fairly useless if somehow stolen. I've gone into this in more detail in [the README](https://github.com/classabbyamp/mkinitcpio-tailscale/blob/master/README.md#security-considerations).

Using this hook with ZFSBootMenu is fairly straightforward.
First, [set up remote access](https://docs.zfsbootmenu.org/en/v2.2.x/guides/general/remote-access.html) (following the mkinitcpio-specific instructions) and set up [Tailscale ACLs](https://tailscale.com/kb/1018/acls/) as you see fit.
I've gone with a setup like this:

```json
// Example ACLs for mkinitcpio-tailscale and ZFSBootMenu
{
	"tagOwners": {
		"tag:zfsbootmenu": ["autogroup:admin"],
		"tag:server":     ["autogroup:admin"],
		"tag:local":      ["autogroup:admin"],
	},

	"acls": [
		{"action": "accept", "src": ["tag:local"], "dst": ["*:*"]},
		{"action": "accept", "src": ["tag:server"], "dst": ["tag:server:*"]},
	],
}
```

Because there is no listed ACL with `tag:zfsbootmenu` in the `src` field, it cannot initiate any connections to other parts of the Tailnet.

Then, generate an [auth key](https://login.tailscale.com/admin/settings/keys) and save it somewhere like `/tmp/mk-ts-authkey`. I recommend creating the key as **not** reusable, **1 day** expiration, **not** ephemeral, and tagged with the relevant ACL tag.

![Tailscale auth key generation dialog, with settings shown as described](/blog/zfsbootmenu-tailscale/auth-key.png)

Next, install [mkinitcpio-tailscale](https://github.com/classabbyamp/mkinitcpio-tailscale) (it's packaged for Void Linux and [maybe other distros](https://repology.org/project/mkinitcpio-tailscale/versions)) and use the provided script to authenticate to Tailscale:

```
# mkinitcpio-tailscale-setup -k /tmp/mk-ts-authkey
```

Once it runs successfully, you should see something like this in the Tailscale [admin console](https://login.tailscale.com/admin/machines):

![Tailscale admin console machine entry for 'void-zbm-test-mkinitcpio' with the tag 'zfsbootmenu' and expiry disabled](/blog/zfsbootmenu-tailscale/admin-console.png)

Add the hook to the `HOOKS` array (`HOOKS+=(tailscale)`) in `/etc/zfsbootmenu/mkinitcpio.conf` after any network setup hooks and run `generate-zbm` to regenerate the ZFSBootMenu image with Tailscale support.

You should now be able to reboot and ssh into ZFSBootMenu using the Tailscale IP or hostname.

![terminal window SSH'd into a test VM over tailscale, showing the ZFSBootMenu interface](/blog/zfsbootmenu-tailscale/zbm-ssh.png)
