---
layout: post
title:  "Emacs + EDTS + (big)CouchDB... a journey"
date:   2014-05-14 21:05:46
categories: tech
tags: emacs edts couchdb bigcouch erlang
---


If you use Emacs to write Erlang, you might have heard of
[EDTS](https://github.com/tjarvstrand/edts). It's a recent Erlang
development suite for Emacs, and it has some impressive features, but
it can be a beast to get setup and working properly with CouchDB and
the BigCouch merge branch.

Current CouchDB master's folder hierarchy is non standard for Erlang
applications, and as a result it does not work well with EDTS. There
is work being done in the BigCouch merge branch and the RCouch merge
branch to restructure things in a standard OTP compliant
hierarchy. This guide is for Emacs + EDTS + the
[BigCouch merge branch](https://github.com/apache/couchdb/tree/1843-feature-bigcouch).

If you just want to load a vanilla VM with no running applications,
then just follow the standard EDTS config, but that's boring. This
guide will get you setup connecting Emacs/EDTS to your local BigCouch
dev cluster, and provides all the nice EDTS features like automatic
code reloading in the running VM, Emacs repl with tab completion and
documentation, show callers and everything else in EDTS.


*WARNING* this is quite a hack, and involves a number of changes and
 updates. I hope that by writing this up I'll get some feedback on how
 to do things cleaner, but it works, and that's the important part ;-)


## Steps

We'll need to update the EDTS start script, make an .edts config file,
and use a hacked together function for spinning up the repl.

Also worth noting is that this setup expects you to manually run
`./rel/boot_dev_cluster.sh` in the BigCouch branch to spin up the
VM. If you forget to do that, EDTS will spin up a VM for you and then
when you try to boot the dev cluster you'll get errors from trying to
run multiple Erlang VMs with the same name.

### Update EDTS start script

We need to update the base EDTS VM start script to use `-name` instead
of `-sname` so that it can talk to the CouchDB nodes. I also like to
add `-hidden` so that th EDTS VM doesn't show up in the list of nodes
in the three node dev cluster. Here's my diff to the [start script](https://github.com/tjarvstrand/edts/blob/master/start):


{% highlight diff %}
diff --git a/start b/start
index de68605..c744e72 100755
--- a/start
+++ b/start
@@ -27,8 +27,9 @@ PLUGIN_DIR=$EDTS_HOME/plugins

cd $EDTS_HOME
exec $ERL \
-    -sname edts \
+    -name edts@127.0.0.1 \
     -edts project_data_dir "\"$PROJDIR\"" \
     -edts plugin_dir "\"$PLUGIN_DIR\"" \
     -pa $EDTS_HOME/{lib,plugins}/*/ebin \
-    -s edts_app
+    -s edts_app \
+    -hidden
{% endhighlight %}

### Add a .edts config

EDTS works by having a .edts config in the top level of your
project. We need to let the config know where to find all the
projects, and we also need to specify the name of the VM. It's
important that the name matches the name of the local dev cluster node
you want to connect to, as EDTS will connect to the existing VM we
start with the dev script, rather than starting a new VM. This way any
changes you make will be immediately reflected in the running VM,
allowing you to curl again or work in remsh without having to manually
reload the modules or dev server. Here's my config, it's pretty
simple:

{% highlight common-lisp %}
:name "dev1"
:lib-dirs '("src")
{% endhighlight %}

Now when you open a `.erl` file within the CouchDB source folder, EDTS
will start and then you will connect to the running dev server. At
this point you'll be able to save files and have them loaded in the
VM, but you won't have a repl available to you. For some reason EDTS
does not start a repl for nodes in remsh's into.

### Opening an inferior Erlang remsh

I cobbled together a little function to spin up an inferior Erlang
shell and remsh into the dev cluster. This works, and let's you
interact with the running VM, but you lose out on all the niceties of
the EDTS shell, like tab completion and inline documentation
boxes.


{% highlight common-lisp %}
;; adapted from: https://github.com/adbl/tools-emacs/blob/master/david.emacs
(defun erlang-shell-connect-to-node (name)
  (interactive "MNode name to connect to: ")
  (let* ((inferior-erlang-machine-options
          (list "-hidden"
                "-name" (format "emacs-remsh-%s" name)
                "-remsh" (format "%s@127.0.0.1" name))))
    (erlang-shell-display)))
{% endhighlight %}

### Hacking together an EDTS remsh

I decided I really wanted to have the proper EDTS repl, so I dove in
and hacked one out today. I updated
[edts-shell.el](https://github.com/tjarvstrand/edts/blob/master/elisp/edts/edts-shell.el)
with a new hardcoded function to connect to dev1 through a remsh. It's
ugly, hardcoded, and it even throws an error at you when you run it,
but it works!

{% highlight common-lisp %}
(defun edts-shell-dev1 (&optional pwd switch-to)
  "Start an interactive erlang shell."
  (interactive '(nil t))
  (edts-ensure-server-started)
  (let*((buffer-name "*dev1-shell")
        (node-name   "dev1@127.0.0.1")
        ;; (command     (list "erl" "-name dev1-remsh -hidden -remsh" node-name))
        (command (list "/Users/russell/src/dotfiles/emacs.d/plugins/edts/start_dev1"))
        (root        (expand-file-name (or pwd default-directory))))
    (let ((buffer (edts-shell-make-comint-buffer
                   buffer-name
                   node-name
                   root
                   command)))
      (edts-init-node-when-ready node-name node-name root nil)
      (when switch-to (switch-to-buffer buffer))
      buffer)))
{% endhighlight %}

I was having trouble properly passing the Erlang VM command line
options, so I switched to using a little shell script to start the
VM. Ideally the commented out command declaration should do the trick,
but I was having issues with it. Here's my start_dev1 script:

{% highlight bash %}
#/usr/bin/env bash

exec /usr/local/bin/erl -name dev1-remsh -hidden -remsh dev1@127.0.0.1
{% endhighlight %}

Hopefully someone has a cleaner approach for this, but it works for
now.

## Results

With this in place I've got a solid Emacs + EDTS dev environment for
hacking on (big)CouchDB. If you have any feedback or suggestions,
please don't be shy as this is definitely a work in progress and I see
it as an minimal working version, not a finished configuration. I
would also like to better understand interacting with running VMs with
EDTS, it seems to be designed with the assumption you're working
locally with a VM started by EDTS, not a running environment. I'm
curious if the author of EDTS has any suggestions on better approach
here.

## Gotchas

Some random things to keep in mind.

### Single node interaction only

Right now you connect to an individual node, in this case dev1. The
local dev cluster spins up dev2 and dev3 as well, but this does not
connect to those nodes, and does not load code with `nl` to upgrade
all nodes with your modifications. I would like to fix this to at
least do an `nl`. You'll occasionally run into whacky behavior on the
dev cluster, just restart it and run `M-x edts-project-node-init` to
reconnect.

### Disconnected from dev1 VM

If you restart the dev server, you'll lose the connection from EDTS
and will get an error when it tries to push your changes. I just do
`M-x edts-project-node-init` to get back up and running.

### Show source under cursor doesn't go into Erlang/OTP source

I was having issues following code into the full OTP source, and I
tracked this down to the compile info stored in the beams pointing to
a directory that did not exist on my system. I'm not sure what
happened there, I was using erlbrew but ended up going back to erln8
and now I can follow code down into the Erlang/OTP source.

### Built in manual

Don't forget to generate the manual with `M-x edts-man-setup RET` so
you can go directly to the man pages inside emacs and also get inline
function docs, very handy!
