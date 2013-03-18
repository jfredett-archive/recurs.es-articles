---
layout: post
title: "Three Wrongs make me Syck"
date: 2012-03-14 10:53
comments: true
categories: ruby, yaml, neo4j, json
---

## In which I track down a bug to three causes

I recently had to put this comment in a codebase for a client, to explain why
I'd written a task that did this:

```ruby

    task :hack_yaml_parser do
      YAML::ENGINE.yamler = 'syck'  
    end

```

The text of the comment follows:

--------------------------------------------------------------------------------

This dodges a bug with httparty and neo4j and psych/syck. The rundown is thus:
 
Neo4j uses a bareword response for certain querys, which is valid JSON
in the world of the ECMA people, but not in the world of the RFC.

HTTParty can't use the superior MultiJson library, because it
interprets (with some force) the RFC version, and thus can't handle the
response. Therefore, it depends on the older, crustier "crack" JSON
library.

Crack uses a -- unique -- approach to parsing JSON. It munges the text
to be valid yaml, then defers it's parsing to the ruby stdlib's yaml
parser.

Ruby upgraded it's YAML engine to "Psych", which is based on libyaml
and very snappy.

Unfortunately, some of the YAML that crack generates is less than
perfect, so sometimes YAML.load throws a Psych::SyntaxError. Due to
what I can only describe as an extremely unfortunate mistake,
Psych::SyntaxError derives from SyntaxError, which follows up the chain
to ScriptError, Exception, and Object.

Astute readers of Avdi Grimm's book "Exceptional Ruby" will note that
this means it's not a subclass of StandardError, which means you cannot
rescue from it (because it is in the same class hierarchy as an actual
ruby syntax error). 

So. You are left with an unrescueable syntax error. You can't use the
parser that would dodge this because it causes worse issues (eg, you
can't get single properties from a node). You can't use the provided
library because it can't parse without error. What's a hacker to do?

Well, you can revert to the older "Syck" yaml parsing library. Which
happily accepts the failing cases, and generates _rescueable_ errors
when it hits a YAML syntax bug.

Thus, this unfortunately terrible line of code is present to fix a bug
that all boils down to someone deriving from the wrong class, someone
else following the wrong rules, and a third person (you) being in the
wrong place, at the wrong time.

--------------------------------------------------------------------------------

So let this be a lesson, follow RFC's, not communities if you can. Don't derive
from the wrong class hierarchy, and try not to be in the wrong place at the
wrong time.

I spent _way_ to much time on this.
