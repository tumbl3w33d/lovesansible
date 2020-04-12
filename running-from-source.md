## Why you should run ansible from source

No matter whether you are new to ansible or an experienced user of this wonderful tool, you probably installed it using the package manager of your OS or pip. You probably did that, because you always choose to install packaged versions of software. They are _stable_, they get updated when you run system updates, you don't want to spend time finding a version that works for you... it just feels right to go that way.

I'd agree with this and it's how I also treat most of the software I use. However, ansible I treat differently. In contrast to a lot of other tools I use, it does not require any compilation of binaries. It is written in Python, a scripting language that gets interpreted on execution. If you change any of the ansible scripts and execute your playbook or ad-hoc command, you will immediately see the effect of your change.

### What are the benefits of running from source?

You can...

![pro](https://github.com/tumbl3w33d/lovesansible/raw/master/images/pro.png "pro")switch back and forth between versions

![pro](https://github.com/tumbl3w33d/lovesansible/raw/master/images/pro.png "pro")browse the code to get deeper insight

While the latter is only relevant to users with understanding of code, the first results in several advantages for every user. Not only can you run the latest available (unreleased) version and benefit of all the new features, but you also have the option to step backwards in the development history and return to a version before a grave bug was introduced. This doesn't take you any effort or time, but can be done in seconds by using a single simple git command:

`git checkout <hash/tag/branch>`

### Wait, git? What?!  (skip if  you know git)

Alright, you might not know git, but you probably heard about versioning of code, version control, source code management,... There are several tools available for this purpose. Ansible chose [git](https://en.wikipedia.org/wiki/Git) for tracking changes which means you also need a git client installed to benefit of the aforementioned advantages.
To keep it short and simple - whenever a developer changes ansible's code, they create a commit. This commit marks a certain state of the code and allows every user of the repository to recreate exactly this state.

### So what are the downsides?

![con](https://github.com/tumbl3w33d/lovesansible/raw/master/images/con.png "con") initial setup more difficult than running `<package manager> install ansible`

![con](https://github.com/tumbl3w33d/lovesansible/raw/master/images/con.png "con") lack of basic git skills nullifies benefits

The installation procedure is described in the [official documentation](http://docs.ansible.com/ansible/intro_installation.html#running-from-source). You mainly need to

1. install git and pip
2. checkout the ansible repository with a version of your choice (if you don't choose any specific version, you will be running the latest development version available at the time of your cloning) 
3. install required python modules
4. load (_source_) a script file in the repository which prepares your shell environment for using ansible.

 This process became even simpler since the core and extra module repositories were merged into the main one.

### Which version should I run?

Running from source is sometimes confused with running _devel_ (the branch on which all the development happens). While you can do that, you are also free to checkout a certain release version and run exactly the version that is maintained as official release.

#### Run (latest) devel

Assuming you already cloned the ansible repository as described in the [official documentation](http://docs.ansible.com/ansible/intro_installation.html#running-from-source):

* `git checkout devel`
  * this makes you run your own copy of the development branch
* `git pull`
  * this updates your own copy of the development branch with the latest changes

It is up to you to decide if/when you update your version. If you don't pull, you will not experience any changes, neither negative (regressions), nor positive (new features, bugfixes). The longer you wait with pulling, the more changes will happen and the harder it gets to find the _bad_ commit that causes a regression that impacts your playbook runs.

Updating often (e.g., daily) has the benefit that you receive only a hand full of changes. If you notice a regression, you can easily go back a few versions and maybe even identify, which commit exactly causes problems. This is your chance to contribute to making ansible even better for all users, simply by creating an [issue](https://github.com/ansible/ansible/issues) and mentioned that you tracked down the evil commit. That makes it easier for the developers to provide a quick bugfix. I've had several cases where the ansible developers provided me with a fix within minutes or at least hours after I created an issue.

#### Run a certain release

Assuming you already cloned the ansible repository as described in the [official documentation](http://docs.ansible.com/ansible/intro_installation.html#running-from-source):

- choose an [active branch](https://github.com/ansible/ansible/branches)
  - the release branches are called `stable-<version number>`
- `git checkout stable-2.3`
  - this makes you run your own copy of the stable-2.3 branch
- `git pull`
  - this updates your own copy of the stable-2.3 branch with the latest changes (bugfixes, backports)

#### Go back in time when a regression happens

Now let's assume you run devel and after a `git pull` things go south. Your playbooks break, ansible doesn't pick up the right variables, doesn't find the right hosts, ignores blocks, is unable to connect...

Whenever you decide to pull to update your current branch, make sure you write down your current version before. There are different ways to do that:

``` bash
$ git log -1
commit ea7bff4a3f35d5ddca5e6f39ff88a5d7cbf60740
Author: Brian Coca <brian.coca+git@gmail.com>
Date:   Tue Mar 28 15:55:09 2017 -0400

    changed spec to options as per irc meeting

```

This shows you the latest commit you have currently checked out. Next to a message that describes what was changed, you see the commit author's name and when it was committed. What is most interesting for you is the commit hash in the first line. In this example that's `ea7bff4a3f35d5ddca5e6f39ff88a5d7cbf60740` This is an alphanumeric string which uniquely identifies this commit, i.e., no matter where in the repository history you currently are working, you can always return to this point by executing `git checkout ea7bff4a3f35d5ddca5e6f39ff88a5d7cbf60740`.

An alternative way to figure out your current version is the following command:

``` bash
$ ansible --version
ansible 2.4.0 (devel ea7bff4a3f) last updated 2017/03/28 22:53:03 (GMT +200)
```

Here you see a shortened version of the hash: `ea7bff4a3f`. It is still long enough to guarantee being unique and you will also find these shortened hashes when browsing the commits on [github](https://github.com/ansible/ansible/commits/devel).

Now let's say you pull to get the latest and greatest stuff from the repository:

``` bash
$ git pull
remote: Counting objects: 868, done.
remote: Compressing objects: 100% (36/36), done.
remote: Total 868 (delta 402), reused 386 (delta 385), pack-reused 443
Receiving objects: 100% (868/868), 263.41 KiB | 0 bytes/s, done.
Resolving deltas: 100% (558/558), completed with 214 local objects.
From https://github.com/ansible/ansible
   ea7bff4a3..2d8c5e6b8  devel            -> origin/devel
   2273800f7..df3e71a83  stable-2.2       -> origin/stable-2.2
   da10768b1..401f6d68d  stable-2.3       -> origin/stable-2.3
 * [new tag]             v2.2.3.0-0.1.rc1 -> v2.2.3.0-0.1.rc1
 * [new tag]             v2.3.0.0-0.3.rc3 -> v2.3.0.0-0.3.rc3
Updating ea7bff4a3..2d8c5e6b8
Fast-forward
[...]
```

The longer you waited since the last time you pulled, the bigger the output will be. Let's assume this version is broken and you have either no time or no motivation to figure out what exactly broke. You just want to go back to what was working for you before, execute your playbooks and get home in time for dinner.

There are (at least) two options to return to the working version:

* `git reset --hard ea7bff4a3f`
  * brings your own copy of the current branch back to this version
* `git clean -df`
  * removes possible leftovers (i.e. files that were added in later commits)

The other option is

* `git checkout ea7bff4a3f`

  * this brings you to a kind of temporary branch that reflects exactly the version you want to run

  * the output of this command might confuse you

    ```bash
    $ git checkout ea7bff4a3f
    Note: checking out 'ea7bff4a3f'.

    You are in 'detached HEAD' state. You can look around, make experimental
    changes and commit them, and you can discard any commits you make in this
    state without impacting any branches by performing another checkout.

    If you want to create a new branch to retain commits you create, you may
    do so (now or later) by using -b with the checkout command again. Example:

      git checkout -b <new-branch-name>

    HEAD is now at ea7bff4a3... changed spec to options as per irc meeting
    ```

  * you can always return to other branches with a simple checkout, e.g.,
    `git checkout devel`

### Summary and plea

I tried to show the benefits and downsides of running from source and demonstrated the necessary commands which are necessary to benefit from this approach. What I only mentioned on a sidenote is the aspect of [contribution](http://docs.ansible.com/ansible/community.html). Open source projects depend on the community. While ansible will still receive improvements and further developments without your or anyone's contribution, it will for sure become a better tool with better user experience and less bugs when you help the developers. It is not necessary to write code to achieve that. Finding bugs and writing precise bugreports helps a lot, because - what happens if no one tests the latest developments? A lot of bugs will not be detected before the regular stable release and will make it into the release branch. The result are bugs that linger in the wild for ages and those users who are really forced to run a version as stable and tested as possible, might be stuck with them for weeks.

So consider your readiness to take a _risk_ and run an unreleased version. I've done that for more than a year, updating approximately biweekly. I had no severe problems and needed to stay at an earlier version than latest only two times. In both cases I received bugfixes within hours and could return to the latest version again.