# GIDDYUP!

Does deploying your application seem like a bit too much of a chore?  Do you
wish it wasn't so hard?  Well, with giddyup it can become just a tiny bit
easier.

If you've ever used, or seen, Heroku's git-based deployment model, you know
how simple app deployment *can* be.  While we don't try to emulate all of
Heroku's excellent infrastructure, giddyup does handle a small corner of
that -- the ability to deploy your app with a simple 'git push'.

Giddyup is not a full-featured deployment tool -- if you do multi-machine
deployments, or you need to do excessively complicated things with your
deployments, then there's a good chance that Giddyup isn't for you.  We're
going after the deployment "low hanging fruit".


# Initial Setup

1. Create a directory somewhere, as the user who will run the app,
   which will serve as the "root" of the whole application deployment.
   > e.g. `mkdir /home/deployuser/appname`

2. Create a directory in the directory you made in step 2, named
   'repo'.
   > e.g. `mkdir /home/appuser/appname/repo`

3. Run git init --bare in the 'repo' directory you created in
   step 3.
   > e.g. `git init --bare /home/appuser/appname/repo`

4. Set the git config variables in the config for the repo you
   created in step 4 (see the section "Configuration" for the
   available variables)
   > e.g. `git config -f /home/appuser/appname/repo/config giddyup.environment
   > production`

5. Symlink the `update-hook` script in this distribution to `hooks/update`
   in the repo you created in step 3 (and make sure `update-hook` is
   executable)
   > e.g. `chmod a+x /path/to/giddyup/update-hook; ln -s
   > /path/to/giddyup/update-hook /home/appuser/appname/repo/hooks/update`

6. Add the necessary hooks to your application's local git repo to effect
   proper deployment.

7. Add the newly created git repo as a remote, then push to it:
   > e.g. `git remote add deploy appuser@example.com:appname/repo;
   > git push deploy master:master`

8. Configure your webserver to pass requests to the appserver, and you
   should be good to go.


# Deployment tree structure

Giddyup creates a tree of directories underneath the specified "root"
directory, that contains (hopefully) everything related to your application. 
The structure is as follows:

* `repo`: This is the repository to which git pushes are made, and within
which the Giddyup hook script is placed.  It must be a "bare" git repo
(because otherwise Git gets ridiculously confused and annoyed) and it is the
parent of this directory which is considered to be the "root" of the entire
application.  (Note: this doesn't *actually* have to be named `repo`; but
it's best to keep a consistent convention, for the sake of sanity).
* `shared`: All of your application's data which should persist between
releases should be kept in here -- log files (if you want to keep the same
log files between releases), **definitely** your customers' uploaded assets
-- all that sort of thing goes in here, and should be symlinked from your
release.  See the section "Shared data", below.
* `releases`: This is where each individual release goes.  Each deployment
of your application gets a directory of it's own in here, named after the
current date/time at the time of deployment, in the format
`YYYYMMDD-HHMMSS`.
* `current`: A symlink to the currently running version of your application.


## Shared data

Anything you need to be shared across deployments should live under the
`shared` directory, and symlinks placed in your releases to point to the
relevant data in `shared`.  You share things in your deployments by calling
the `share` helper function within your hook scripts.
This function will take a relative path (relative to the root of your repo)
and create a symlink into the same path within `shared`.

For example, you might have a hook script that uses Bundler to setup your
local gems.  Since you don't want to have to dick around reinstalling all
your gems with every release, you want to share that bundle between
releases.  Your hook script might look something like this:

    . /path/to/giddyup/functions.sh
    share vendor/bundle
    cd "${RELEASE}"
    bundle install --deployment

This will create a symlink from `$ROOT/releases/<timestamp>/vendor/bundle`
to `$ROOT/shared/vendor/bundle`, then run bundle in the new release to
ensure that all your required gems are available.

If a file or directory already exists in `releases/<timestamp>` that is
supposed to be symlinked, we'll remove it before creating the symlink.  The
directory structure in `shared` will always mirror that of your releases
when you use `share`; this improves simplicity and comprehensibility.  It
really isn't worth customising; trust me.

If a symlink destination doesn't exist already within `shared`, then we'll
try to infer whether it's a file or directory from the type of the data
already existent in the deployed code.  Leading directory components will be
created in `shared` automatically, as well.


# Hooks

Because everyone's application is slightly different, Giddyup itself doesn't
actually do very much itself -- instead, it delegates a lot of things back
to code that you write, as hooks.  To understand hooks, it's best to
understand how Giddyup works:

1. You push your updated code to the repo controlled by giddyup
2. The `update` hook, provided by Giddyup, starts to run
3. Giddyup makes a copy of the code in your deployment repo
4. Giddyup runs the `stop` hook
5. Giddyup changes the "current" symlink for the deployment to point at the
newly configured code
6. Giddyup runs the `start` hook
7. Old releases are tidied up, if required (see the `giddyup.keepreleases`
config variable)

To put it another way, the following hooks are available:

* **stop**: Run after the new release is in place, but while the system's
  idea of the "current" release still points to the currently running code. 
  You'll probably want to do whatever's required to stop your appserver from
  running in this hook, run bundler, and perhaps put up a maintenance page.
* **start**: Run after the "current" symlink has been changed to point to
  the new code.  In here you'd probably want to do database migrations,
  start your appserver, and take down your maintenance page.


## Running hooks

Giddyup runs hooks by looking in the **newly deployed** version of your
application, in the location specified by the `giddyup.hookdir` git config
variable (see "Configuration", below).  It is looking for files or
directories that match the name of the hook (`start`, `stop`, etc).  If
there is a file named after the hook, and it is executable, then that file
is executed.  If, on the other hand, there is a directory named after the
hook, then all executable files in that directory are executed, in the
lexical order of the names of the files.

Each hook script is run as a separate process, and as such cannot effect the
environment or working directory of giddyup itself or any other hook script.


## Hook environment

The environment of the hook is very minimal; only the following environment
variables will be set:

* `PATH` -- the same as the path of the Giddyup script itself.
* `RACK_ENV` -- the environment specified by `giddyup.environment` (see
"Configuration", below).
* `ROOT` -- the directory that is the "root" of the entire deployment; the
directory which the deployment git repository (and everything else) lives
* `RELEASE` -- the canonical directory that contains the data in this
release of the application.
* `NEWREV` -- The SHA of the commit which is being deployed.

The working directory of all hooks is the root of the deployment tree. 
During the 'stop' hook, the `current` symlink will point to the previous
running release of the application, while during the `start` hook the
`current` symlink will point to the new release you're currently deploying.


## Hook script helpers

To help you make your hook scripts easier to write, there are some shell
functions available to help you on your way.  To use them, merely add:

    . /path/to/giddyup/functions.sh

at the top of your hook script, and then call away to your heart's content.



# Error handling

At present, the error handling in giddyup is a bit primitive.  In general,
if any part of the giddyup script fails, the whole process will abort.  This
*mostly* won't hurt your running app, because giddyup does as much as it can
before starting or stopping anything.  However, if your hook scripts bomb
out, or one of the things that runs between stopping and starting the app
fails, then things could be left in a bit of a limbo state.

Improvements to this part of the system, particularly around intelligent
rollback strategies, is planned.  Patches welcome.


# Configuration

Gidduyp is controlled by git repository configuration variables; these can
either be set using `git config` in the deployment repository, or by editing
the `config` file in the repository directly.

Available configuration variables are given below.


## `giddyup.environment`

(**REQUIRED**)

Every Rack application needs to have an environment name to run in. 
`giddyup.environment` specifies the name of the environment this app runs
in.  Since this environment is specified per-repository, you can run several
environments (say, uat, staging, and production) with the same in-repo
deployment hooks, just by pushing to different remote repositories.


## `giddyup.hookdir`

(**OPTIONAL**; default `config/hooks`)

The directory within which Giddyup will look to find hook scripts, relative
to the root of your working copy.  The intent is that the hooks relating to
the deployment of your application are best kept *with* your application.


## `giddyup.keepreleases`

(**OPTIONAL**; default: 5)

Giddyup will keep a few older deployed releases available, in case you need
to make an emergency rollback, or examine exactly what was recently
deployed.  This configuration parameter determines how many releases to
keep.  Set this to 1 to not keep any previous releases, and set it to 0 if
you want to keep every release ever (not a good idea unless you've got one
of those fancy new infinite-capacity hard drives).


## `giddyup.debug`

(**OPTIONAL**; default: false)

If you want to have far, far too much gory detail about what giddyup is
doing, you can set this to true.


# Frequently Answered Questions

## What?  Is that all?

Yes, the main giddyup update hook script is quite minimal.  The aim here is
to provide the necessary scaffolding upon which customised deployment
processes can be implemented.

To put it another way: yes, Giddyup is small and dumb.  You're supposed to
provide the smarts in hooks.  If you have a general-purpose hook you'd like
to share with others, please feel free to [send it to
me](mailto:theshed+giddyup@hezmatt.org) and I'll include it in the examples.


## I need moah hooks!

Weeeeell... maybe you do, maybe you don't.  Remember that you can execute as
many different hook scripts as you like, and in the order you specify.  For
example, you might think you need a "pre-stop" hook, to do things that might
take a while, before the application is stopped (to minimise downtime).  You
don't actually need a separate hook in giddyup; all you need to do is move
the hook script that actually stops your appservers to after the hook script
that does whatever time-consuming task you have in mind.
