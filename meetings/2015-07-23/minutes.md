# Node.js post-mortem diagnostics meeting TBD

Meeting recording: https://www.youtube.com/watch?v=8baN-X60H6M

## Present

Howard Hellyer (@hhellyeribm) - IBM
Chris Bailey (@seabaylea) - IBM
Bradley Meck (@bmeck ) - Nodesource
Joshua M. Clulow (@jclulow) - Joyent
Dave Pacheco (@davepacheco) - Joyent

## Agenda

1. Governance - review boilerplates and agree/disagree to use for this project
  + https://github.com/nodejs/post-mortem/blob/master/CONTRIBUTING-draft.md
  + https://github.com/nodejs/post-mortem/blob/master/GOVERNANCE-draft.md

2. Discuss/Confirm scope we want to tackle
  + Define end goals, initial ideas include:
   + Defining and adding interfaces/APIs in order to allow dumps to be generated when needed
   + Defining use cases for post mortem analysis (general debugging, memory analysis)
   + Defining types of dumps and other post-mortem data to  be generated/collected
   + Defining and adding common structures to the dumps generated in order to support tools that want to introspect those dumps

3. Identify initial next steps

## Minutes

1. Agreed to use standard governance rules.

2. General discussion around the purpose of the group.
Discussed need to define an end goal and what we are trying to achieve for the Node.js community. We should consider promotion of post-mortem tooling possibly in the primary Node.js documentation.

Discussed whether some of the areas we would be interested in were more related to v8 than Node.js directly.
v8 team believed to be mostly interested in debugging from JSON artifacts not from core files. Discussed core dumps versus v8 heap dumps.

3. The next steps will be to raise issues to catalogue existing tools and use cases, types of users and identify gaps. What are the target audiences of this group.

AOB - Confirmed group was happy with the term post-mortem to describe this area. Collectively agreed this was a standard term.

## Next Meeting
Issue to be raised to discuss date of next meeting.
