# Node Post-Mortem WG Meeting Notes - July 18 2016

* Recording here: http://www.youtube.com/watch?v=HIh7kQUifuU
* Issue link: https://github.com/nodejs/post-mortem/issues/31

# Present

* Yunong Xiao @yunong
* Dave Pacheco @davepacheco
* Joshua M. Clulow @jclulow
* Michael Dawson  @mhdawson
* Richard Chamberlain @rnchamberlain


# Agenda

* Stand up
* Actions from last meeting
* Where to put code we are collaborating on
  https://github.com/nodejs/post-mortem/issues/30
* Javascript API to support common extensions between MDB/lldb/IDDE
* NodeReport as module bundled into Node ?

# Standup

* Yunong Xiao @yunongx
  * working on talk for post-mortem WG


* Dave Pacheco @dapsays
 * Met with Yunong to talk about background info for somebody working on mdb/v8


* Joshua M. Clulow @jmclulow
 * no time


* Michael Dawson  @mhdawson
  * working on talk for post-mortem WG
  * Working with Richard on a his efforts in post-mortem,
    mostly on the brainstorming direction side as opposed to commits


* Richard Chamberlain @rnchamberlain
 *  working with Howard Hellyer on NodeReport and llnode/lldb contributions


# Actions from last meeting
N/A

# Agenda Item Disussion

## Where to put code we are collaborating on

https://github.com/nodejs/post-mortem/issues/30

We discussed the different options. No objection to products going under
github/nodejs and this is the first choice from the options, but this
should be optional.  ie. only when it makes sense for
the project and the foundation. Some projects will be under github/nodejs
but others will be external.

Just as important is making sure that people can find all the tools and
projects that this WG is working on.  First step is to update the readme.md
to add a section for this with additional links.  Following that we may
want to brainstorm other ideas of how to get the message out.

ACTIONS:
  * Michael to create PR on readme.md that people will comment on to add
    the links/projects
  * Michael to capture consensus discussed in issue #30 and then
    add to CTC agenda to start discussion to see if we can move


## Javascript API to support common extensions between MDB/lldb/IDDE

https://github.com/nodejs/post-mortem/issues/33

* Richard did quick overview of concept.
* Has done quick prototype.
* David, sounds cool, but largish effort for mdb. Their path might be
  to finish common core dump generation file format.  Then have API use
  that file to implement API.  Other debuggers could then also implement
  the API directly if they want to.
  We can then collaborate on the commands using that API.
  Nice to be able to have 2 bases to develop API one being the common format.
* Yunong - have we finalized the common format ? not yet.
  Is one of the things we need to close one.
* There is an issue for the common format. Next is to prototype,
  Dave believes he could have that for the next meeting.
  Prototype would be command from mdb/v8 which would generate
  common dump format.  Richard to look at generating common dump
  format using lldb.

ACTIONS:
* David plans to have generation of common format with mdb done for
  next meeting.
* Richard to look at generating common format with lldb and report
  back on that.
* Richard to do initial cut at what API might look like.

## NodeReport as module bundled into Node

Michael's question is if npm for NodeReport should be part
of the Node distribution?

Mdb was at one point it was bundled with but that ended up being removed.

There have been tensions over this in the past in npm,
with the npm org wanting different consumption
level than Node team wanted.

Yunong - list off the tradeoffs on bundling or not bundling.

Take back to github, discussion of pro/cons

How is this different from any other modules

Michael, feel that you should get at least minimal capability with
runtime without having to install anything else.  Sometimes need
legal reviews/ok to add something to production.

Yunong, they might tend towards not bundling as they want
to fine tune what they use.

Take back to github, discussion of pro/cons

ACTIONS:

* Michael to open issue for discussion.

## Other issues

Get together at Node Summit ?  - Monday agreed.
