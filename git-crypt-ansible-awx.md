# Using git-crypt with ansible

I want to describe my experiences with trying out `git-crypt` in combination with `ansible` and give some hints for those who might be interested, too.

- [TOC](#using-git-crypt-with-ansible)
  * [Why would you do that when there is ansible-vault?](#why-would-you-do-that-when-there-is-ansible-vault)
    + [The long story](#the-long-story)
      - [Motivation #1 - heterogeneous repos](#motivation-1---heterogeneous-repos)
      - [Motivation #2 - people are people](#motivation-2---people-are-people)
    + [TL;DR](#tldr)
  * [Necessary adjustments for use with AWX/Tower](#necessary-adjustments-for-use-with-awxtower)
    + [Storing the symmetric key in AWX](#storing-the-symmetric-key-in-awx)
  * [Conclusion](#conclusion)

## Why would you do that when there is ansible-vault?

### The long story

#### Motivation #1 - heterogeneous repos
Having used ansible for many years, I have, of course, been using ansible-vault a lot and did not even think that I would ever need an alternative. Especially when it was enhanced with the feature that allows you to use different secrets within a single repository, I was a happy camper utilizing it.

However, recently I had some experiences that made me revisit the approach and I remembered I once heard about a generic tool for encrypting secrets in git repositories, called [git-crypt](https://github.com/AGWA/git-crypt).

The first use case was when I started using [kolla-ansible](https://docs.openstack.org/kolla-ansible/latest/) to install and maintain my private OpenStack cloud. Shoutout to the guys who made it. I love the tool!

`kolla-ansible` is basically a bunch of roles wrapped with a convenience script that accepts commands and in turn generates calls to `ansible-playbook` with all necessary parameters.
I created a dedicated git repository for maintaining the related configurations and also started adding own playbooks for maintenance actions, preparing hosts for the cloud and the basic cloud configuration with virtual networks, projects, users, quota, fixed floating IPs etc.

Now here comes the catch - sometimes you need to interact with the cloud using the related CLI tool `openstack` or low-level plumbing tools like `nova`, `cinder`, etc., e.g., for troubleshooting. This requires secrets for authentication and to make this easily accessible, you don't want to (find and) manually decrypt them everytime you enter the repository. Since these CLI tools aren't connected to ansible at all, you don't benefit of the automagic decryption ansible provides.

The solution - with `git-crypt` you only unlock the repository once after cloning it and from then on, the encryption is transparent for you, i.e., files will be encrypted when pushed to a server, but on your workstation you will see everything in plaintext (which is beautiful for reviewing changes, too).

Of course, you could mix the use of `ansible-vault` and `git-crypt` but when you have `git-crypt` in place with all its comfort, there is no point in going back to staring at AES blocks in files and continuously decrypting them for getting variable names or changing values.

#### Motivation #2 - people are people
The more people we involved in maintaining ansible repositories, the more often it happened that they forgot to (re-)encrypt secrets they added/edited. That's less dramatic when you only work in private repositories with access restricted to few, however, you never know if the repository will still be in private space in a year and git does not forget.

So whenever this happened and someone noticed, we rewrote the git history to cleanup the mess. There are several approaches to this, some more people-intensive, some more radical (sacrificing commit signatures with [a repo cleaner](https://rtyley.github.io/bfg-repo-cleaner/) or history by squashing). It also depends on how many commits ago it happened.

> Don't you have a review process, like pull requests?

Yes, we do. But to get back to the subheading - quality differs, some PRs are bigger than they should be, people are under heavy work load…
but enough excuses. It's easier to change tools than to change people.

I would not call `git-crypt` the savior for all cases here, but since you can base encryption on quite generic patterns like `**/secrets.yml` you can get away with a good hit rate and reduce the chance of leaking secrets. Plus, people can review their own changes easier in a local unencrypted diff before committing/pushing.

### TL;DR
![pro](https://github.com/tumbl3w33d/lovesansible/raw/master/images/pro.png "pro") transparent encryption (i.e., on your workstation you see everything in plaintext and get clear diffs)

![pro](https://github.com/tumbl3w33d/lovesansible/raw/master/images/pro.png "pro") you don't need to remember (re-)encrypting everything before committing/pushing

![pro](https://github.com/tumbl3w33d/lovesansible/raw/master/images/pro.png "pro") repositories that aren't purely ansible-focused, i.e., contain other secrets that wouldn't be automagically decrypted for use

![neutral](https://github.com/tumbl3w33d/lovesansible/raw/master/images/neutral.png "neutral") you need to install an additional tool (but it's probably in the package management of your distro)

![neutral](https://github.com/tumbl3w33d/lovesansible/raw/master/images/neutral.png "neutral") if you have [gpg](https://gnupg.org/) established in your team, you easily share access to repository secrets by gpg key instead of passing a vault password around

![neutral](https://github.com/tumbl3w33d/lovesansible/raw/master/images/neutral.png "neutral") you need to make adjustments if you want to run your repository in AWX/Tower

![con](https://github.com/tumbl3w33d/lovesansible/raw/master/images/con.png "con") only a single key for the entire repository

## Necessary adjustments for use with AWX/Tower

Using `git-crypt` with ansible is quite easy, but when you want to use such repository in AWX, you need to handle the case that the repository is "locked" (encrypted) after AWX clones it to run a job.

It gets even more difficult when a dynamic inventory is involved which needs access to encrypted values, like the API credentials of a cloud provider.

My first approach was to just give AWX an own gpg key and add it as `git-crypt` collaborator. Then I would just run an unlock role first and continue as usual. The same logic could be used within the dynamic inventory code before actually listing the available hosts.
To sum it up - I couldn't make this approach work. I even provided a key pair without password to AWX to start simple, but AWX wouldn't find the key for a reason I couldn't figure out until I gave up. Manually executing the unlocking process of the repo of a running job (that I put asleep for troubleshooting) worked, but something about the environment of a real job execution must be different.

But enough of the approach that didn't work. Fortunately, `git-crypt` offers a second option to unlock a repository. Either way, `git-crypt`'s encryption is based on a symmetric key that can be exported by executing

```
git-crypt export-key /path/to/key
```

and shared among the trusted users of the repository. The gpg option is just built on top of that and avoids sharing the plaintext secret. Instead, the symmetric key is encrypted using the gpg public keys of all trusted users and when they unlock the repository, they really unlock the symmetric key first and then the rest is same for both options.

### Storing the symmetric key in AWX
> So how do you pass this key to AWX for unlocking repositories?

I wasn't too familar with the AWX credential management yet so I thought you are limited to the credential types that it comes with. None of them were generic enough to store a generic key (file).

Then I found out about [credential types that you can define yourself](https://docs.ansible.com/ansible-tower/latest/html/userguide/credential_types.html) and I was amazed by how smart and clean this was done on their end. Long story short - here comes the solution:

The input definition
``` json
{
 "fields": [
  {
   "id": "git_crypt_key",
   "type": "string",
   "label": "Key",
   "secret": true,
   "help_text": "Symmetric key exported using 'git-crypt export-key'",
   "multiline": true
  }
 ],
 "required": [
  "git_crypt_key"
 ]
}
```

The injector definition
``` json
{
  "env": {
    "GITCRYPT_KEY_PATH": "{% raw %}{{ tower.filename }}{% endraw %}"
  },
  "file": {
    "template": "{% raw %}{{ git_crypt_key }}{% endraw %}"
  }
}
```

Unfortunately, the input is limited to `string` and `boolean` and the validator complained about the bytes I tried to paste in the field. Here `base64` encoding came to the rescue, so I ended up storing the key as encoded block and would need to handle the decoding later when using the key.

Now that the credential was ready for use, I had to figure out how to use it in my dynamic inventory and all I needed was this logic executed before grabbing the API key from the encrypted repository file. Because I didn't want to hurt the overview in my actual dynamic inventory, I decided on creating a wrapper inventory that's just for AWX and not in the inventory path defined in the `ansible.cfg`

``` python
#!/usr/bin/env python

import argparse
import base64
import os
import subprocess
from subprocess import PIPE
import yaml

def _base64_decode_symmetric_key():
    with open(os.environ['GITCRYPT_KEY_PATH'], "rb+") as file:
        byte_key = base64.b64decode(file.read())
        file.seek(0)
        file.write(byte_key)
        file.truncate()

def _unlock_git_crypt():
    try:
        # 1st attempt to read
        with open('group_vars/all/secrets.yml') as file:
            yaml.load(file, Loader=yaml.FullLoader)
    except UnicodeDecodeError:
        # reading failed, consider encrypted, try unlock
        try:
            _base64_decode_symmetric_key()
            subprocess.run(["git-crypt unlock $GITCRYPT_KEY_PATH"],
                           shell=True, check=True, universal_newlines=True, stdin=PIPE, stdout=PIPE)
        except subprocess.CalledProcessError:
            print("Failed to execute git-crypt unlock")
        try:
            # 2nd attempt to read
            with open('group_vars/all/secrets.yml') as file:
                yaml.load(file, Loader=yaml.FullLoader)
        except UnicodeDecodeError:
            print("Unable to unlock this repo")


def dump_host(name):
    inv = subprocess.run(["./inventories/hcloud_inventory.py",
                          "--host", name], check=True, universal_newlines=True, stdin=PIPE, stdout=PIPE)
    print(inv.stdout)

def dump_list():
    inv = subprocess.run(["./inventories/hcloud_inventory.py",
                          "--list"], check=True, universal_newlines=True, stdin=PIPE, stdout=PIPE)
    print(inv.stdout)


if __name__ == '__main__':
    PARSER = argparse.ArgumentParser(
        description='inventory wrapper for AWX')
    PARSER.add_argument('--list', help='list all hosts', action='store_true')
    PARSER.add_argument('--host', help='retrieve single host information')
    ARGS = PARSER.parse_args()

    _unlock_git_crypt()

    # if secrets can be read, just passthrough to real inventory
    if ARGS.list:
        dump_list()
    elif ARGS.host:
        dump_host(ARGS.host)
```

The subprocess calls look a little prettier starting with python 3.7, but my AWX was running with 3.6 at that time and while you could change that by modifying the virtualenv, I felt like that's not a battle to fight at that moment.

Another quirk of this logic that you probably spotted - hardcoded paths. The reason behind the `secrets.yml` path is that for 5y there's a feature request in `git-crypt` [waiting to be implemented](https://github.com/AGWA/git-crypt/issues/69) which would allow to determine whether a repository at hand is encrypted or not. Since this isn't there yet, I needed to come up with an alternative and while you'll find more generic solutions when following this ticket, I opted for trying to load an encrypted file that's available in most if not all my repositories. You might want to enhance that on your own.

The second spot where I have paths hardcoded simply lacks a more generic way of detecting the actual inventories. I might improve that in the future by reading the `ansible.cfg` or using a `find`-type of logic. But since all my repositories are structured similar, this works for me.

Loading the repository in AWX with this logic prepended succeeded and I could go ahead run a job template of a playbook which started with a simple call to `git-crypt unlock $GITCRYPT_KEY_PATH`.

Unfortunately, this failed early with a strange encoding error message about surrogates not being allowed. I realized this was caused by ansible not being able to load the (`git-crypt`-)encrypted `group_vars` file mentioned above because it just contains ugly bytes in encrypted state.

So I searched for a hook in ansible that hits early enough to allow the default var loading to work and I was lucky to find that a [vars_plugin](https://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.html#vars-plugins) can do exactly this. While I couldn't find out about the actual execution order of plugins, I just gave it a shot with this naive approach of a plugin:
``` python

import base64
import os
import subprocess
from subprocess import PIPE

from ansible.plugins.vars import BaseVarsPlugin
import yaml

def _base64_decode_symmetric_key():
    with open(os.environ['GITCRYPT_KEY_PATH'], "rb+") as file:
        byte_key = base64.b64decode(file.read())
        file.seek(0)
        file.write(byte_key)
        file.truncate()

def _unlock_git_crypt():
    try:
        # 1st attempt to read
        with open('group_vars/all/secrets.yml') as file:
            yaml.load(file, Loader=yaml.FullLoader)
    except UnicodeDecodeError:
        # reading failed, consider encrypted, try unlock
        try:
            _base64_decode_symmetric_key()
            subprocess.run(["git-crypt unlock $GITCRYPT_KEY_PATH"],
                           shell=True, check=True, universal_newlines=True, stdin=PIPE, stdout=PIPE)
        except subprocess.CalledProcessError:
            print("Failed to execute git-crypt unlock")
        try:
            # 2nd attempt to read
            with open('group_vars/all/secrets.yml') as file:
                yaml.load(file, Loader=yaml.FullLoader)
        except UnicodeDecodeError:
            print("Unable to unlock this repo")

class VarsModule(BaseVarsPlugin):

    REQUIRES_WHITELIST = False

    def get_vars(self, loader, path, entities):
        _unlock_git_crypt()
        return {}

```

As you can see, it contains the same logic seen before which handles the base64 key and unlocks the repository before doing… nothing else. Fortunately, after activating it in my `ansible.cfg`
``` ini
[defaults]
[…]
vars_plugins = ./plugins/vars_plugins/
```
this was executed before the default var loading and my playbook run succeeded because AWX automagically found an unencrypted repo.

## Conclusion
> Does it work to fully replace `ansible-vault` with `git-crypt`?

Absolutely.

> Is it worth it?

That's something you need to decide for yourself, given the list of advantages and disadvantages and what you saw in this article. In addition, you should read what the [git-crypt documentation says about its limitations](https://github.com/AGWA/git-crypt#current-status).

For me working with `git-crypt` is more comfortable and I will go ahead and turn more of my bigger `vault`-style repositories with many scattered secrets into `git-crypt` ones.