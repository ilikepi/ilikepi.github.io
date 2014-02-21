---
layout: post
title: Chruby, Bundler, and The Search For Binstubs
---

## Moving forward

Recently at work we've been reevaluating our methods for managing rubies and
gems.  [RVM][] and its gemset support have served us pretty well, but its
complexity and rapidly rolling release cycle make it difficult to deal with
problems when they arise.  Cooincidentally, I came across a pair of blog posts
by Stefan Wrobal who has been having similar thoughts to our own.  In his
[first post][swrobel-part-1], he discusses his migration from RVM to
[ruby-install][]+[chruby][].  In [part two][swrobel-part-2], he describes his
bundler workflow.  These are excellent write-ups, and I was quite interested
in adopting his methods save for one minor quibble: relative PATH entries.

## Relative PATH entries

Long ago, when I was first getting to know UNIX, I became frustrated with a
rather innocuous problem.  I regularly wanted to execute a program that
resided in my current directory, but I kept forgetting to add the './' in
front of the program's name.  After a little research, I discovered that by
adding '.' to my PATH variable, I no longer needed this prefix.  If the
program resided in my current directory, I could simply type its name alone
(or more likely just enough characters to tab-complete its name), and the
shell would find it.  A quick change to my shell startup file and that was
that.

A while later, after I leveled up once or twice, I came to understand why this
is an incredibly bad idea.  If '.' represents whatever current directory you
are in, and you add it to your PATH, it becomes quite easy to run a command
from the current directory when this was not your intent.  Further, if you
happened to have added '.' as the **first** entry in PATH, and you happen to
find yourself in a directory containing, say, a malicious `ls` executable, you
could be in for a long night of pain.

With this revelation, I developed a strong distaste for relative entries in
PATH.  Sure, you can argue that './bin' is measurably less dangerous than '.'
alone, and you can label me paranoid for sacrificing convenience to be
imperceptively more secure.  That's fine, I am paranoid about lots of things.
So now my challenge starts to become clear.

## The challenge

Each of our projects has a bunch of gems, and some of those gems have
command-line programs we'd like to run.  Bundler provides one way to do this.
For a given gem 'foobar' with a command-line program `foo`, we can run `bundle
exec foo` to run the program.  That's helpful, but typing `bundle exec` over
and over gets repetetive, so let's look at some options for tidying things up.

#### Shell aliases

Modern command shells (even Windows' PowerShell) allow you to create command
aliases.  We could simply alias the `bundle exec` versions of our common
commands:

```bash
# add to startup file e.g. ~/.bash_profile
alias rails='bundle exec rails'
alias rake='bundle exec rake'
alias cucumber='bundle exec cucumber'
alias rspec='bundle exec rspec'
```

Now when we type `rake db:migrate`, the shell expands the alias and executes
`bundle exec rake db:migrate`.  Great...except now we have to manually
maintain a list of aliases that <strike>may</strike> will change.  Boo-urns.

#### Magical scripting

Chruby includes [auto.sh][chruby-auto] to magically switch rubies when there
is a .ruby-version file present.  That script is pretty simple, so perhaps
with some modification it could do just a bit more work to figure out where
bundler has stashed its binstubs.  Perhaps that modification might look
[something like this][chruby-auto-bundle-bin].

As it turns out, this was not as simple as I'd hoped, more than doubling the
line count of the original `auto.sh`.  I didn't write tests for it either,
since it seems unlikely to ever be merged.  I have been using it for the past
month or so, however, and I've only seen it barf after having its current
directory deleted out from under it.  Still, it's hardly ideal.

#### Klugey scripting

For kicks I experimented with the following bash script (or variations of it):

```bash
#!/usr/bin/env bash

exec bundle exec "$0" "$@"
```

By placing this script in a generic bin/ directory already present in PATH,
and symlinking common gem commands like `rails` and `rspec` to it, these
commands are translated to their `bundle exec` equivants.  I figured it would
be easier to write a little gem that would automatically ensure that these
symlinks existed than to deal with alias maintenance.  And lo, the desired
behavior was achived.  Except in a couple cases where it wasn't...like `rails`
and `rake`, which seem to cause ruby to find its way into an infinite loop.
But no worries, it's not like those are important commands.  Doh...

## Unsatisfying conclusion

Presently, I'm sticking with my hacked auto.sh script, but I suspect it will
need some tweaks.  I believe Rubygems 2.2 has some changes in store for how it
manages binstubs.  Additionally, the `rails new` command in Rails 4 creates a
bin/ directory under Rails.root with a few binstubs of its own.  The search
continues for a cleaner and more elegant solution.

## References

* [rvm implode][swrobel-part-1], Stefan Wrobel
* [Bundle Everything][swrobel-part-2], Stefan Wrobel
* [RVM][] project
* [ruby-install][] project
* [chruby][] project
* [chruby's auto.sh][chruby-auto]
* [my hacked auto.sh][chruby-auto-bundle-bin]


[RVM]: http://rvm.io/
[ruby-install]: https://github.com/postmodern/ruby-install/
[chruby]: https://github.com/postmodern/chruby/

[swrobel-part-1]: http://www.stefanwrobel.com/rvm-implode
[swrobel-part-2]: http://www.stefanwrobel.com/bundle-everything

[chruby-auto]: https://github.com/postmodern/chruby/blob/master/share/chruby/auto.sh
[chruby-auto-bundle-bin]: https://github.com/ilikepi/chruby/blob/auto_bundle_bin/share/chruby/auto.sh
