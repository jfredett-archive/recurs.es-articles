---
layout: post
title: "Bonsai and Elasticsearch and Tire"
date: 2012-04-02 14:32
comments: true
categories: ruby, elasticsearch, bonsai, heroku, tire, git
---

Working on a client project where we use a statically generated ElasticSearch
index. We recently noticed that [bonsai.io](http://bonsai.io/) was available on
[heroku](http://heroku.com/) for free public beta usage. Previously we
self-hosted on amazon. Obviously it's nice to have everything in one place, so
we decided to move.

On the old system, we used [Tire](http://github.com/karmi/tire) as to generate
the indicies and mappings, as well as to generate the correct query strings, and
to wrap the results of such queries into nice structures. Needless to say Tire
was doing some heavy lifting for us. In particular, we used it to bulk-populate
the indices, which made regenerating the indicies on demand very quick, which is
desirable when you're trying to keep your publish times down.

One last important thing, Tire has an opinion about how you index models in your
system -- one index per model. This means that multiple types of index for a
single model is possible -- eg, having `searchserver.com/items/english/<query>`
and `searchserver.com/items/farsi/<query>` -- in the event you wanted to
separate these 'logical' indicies from elasticsearch's notion of a 'literal'
index[1].

Bonsai has another opinion -- one "literal" index per _app_. Naturally, there is
some impedance to overcome. 

To do this, I had to pull in two different changes from two lovely guys on the
githubs. First was [this](https://github.com/karmi/tire/pull/194) pull request,
by [dylanahsmith](https://github.com/dylanahsmith), this implemented the "PUT"
mapping API, which allows for mappings per logical index ('type' in proper ES
terms). That got me halfway there, but bulk upload was also broken due to the
difference of opinion.

So I pulled in changes from [Evan-M](https://github.com/Evan-M) which I found
via [this](https://github.com/karmi/tire/issues/240) issue. This changes how you
configure tire to be aware of how bonsai does it's indexing.

Once I pulled both of these changes in, I ran the tests -- got 4 failures.
Fortunately, they're the same 4 that are on master. To that end, I feel pretty
confident about using it, especially since it is a reasonably non-interacting
part of the app. 

So, if you want to use my combined changes, you can find them
[here](https://github.com/jfredett/tire) on the master branch. Be warned that
these may not be totally operational in every way. Our usecase is pretty small
in scope, so it's use-at-your-own-risk.



[1] the "logical" and "literal" terms are mine. ES calls them "Indexes" and
"Types" -- but I don't think "type" is the right word for the thing.

