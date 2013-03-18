---
layout: post
title: "Lessons Learned: A Sortah Postmortem"
date: 2011-12-16 17:27
comments: true
categories: [post-mortem, code-I-wrote, ruby, sortah]
---

## Ruby is flexible

Flexible like a Russian Gymnast. Seriously, it was never more than a few moments
of thought to bend ruby to my will. Still, there were some wiggly tricks I had
to use to get some things to work, and there still are several things which need
fixing. For instance

``` ruby This doesn't work

    sortah do
     #...

      router :lenses => [:foo] do
        #... 
      end

    end

```

This doesn't work, presently you have to write it with `router :root, :lenses => [...]` 
because the "parser" (mine, and in a way, the ruby parser) doesn't know how to
handle an implicit first argument -- that is, I need positional arguments, and
have to simulate them with defaulted arguments. This isn't a new problem in
ruby, in fact, 2.0 is supposed to provide named arguments, which will make life
a bit nicer. One way I could get around this is to ~~hack~~ \*cough\* enhance the
parser to check the class of the first argument. So it would go from:


``` ruby Current `router` parser from lib/sortah/util/component.rb

    def initialize(name, opts = {}, *potential_block)
      @name = name
      @opts = opts
      @block = potential_block.first unless potential_block.empty?
    end

```

to something like:

``` ruby Current `router` parser

    def initialize(name, opts = {}, *potential_block)
      if name.is_a? Hash
        @opts = name 
      else
        @name = name
        @opts = opts
      end
      @block = potential_block.first unless potential_block.empty?
    end

```

which would allow (I think) for defaulting. It might have issues with how I
extract the optional block, but that would require actually applying this --
which I'm not really willing to do. Mostly because it's ugly, but also because
I'm not sure it's so bad to have to specify that your root router is in fact a
root router when you want to run lenses. I think it makes more sense to _not
have lenses applied to the root router_. Here's my reasoning.

Almost every lens you run is useful in a fairly limited context, For instance, I
may not want to share my spam filters between my personal email and my work
email -- since the types of email I get in my personal email are very different
than the types I get from my work email. This is not something that occurs to
you normally -- what you think is, "I want to filter my email for spam" so
naturally you would tack the lens at the root. The problem is -- this arguement
could pretty much be made for _every_ lens, "Ooh, wordcount, gonna need that" or
"Hey now, definitely going to need the mailinglist name extractor" so you end up
with a pile of lenses on the root router, and suddenly your email sorting is
thrashing the CPU and taking 20 minutes besides. 

The practicality of sortah is this -- routers are much more likely to be cheap
than lenses. Remember that a router _should_ boil down to a few conditionals and
_consumption of existing metadata_, it's job is more or less to _delegate to
other controllers_. In a sense, routers are controllers, lenses are models. The
former should be thin, the latter should be fat. Thinness here means both the
supplied code to do the routing, and the number of lenses attached, should be
small. To understand why, we can look at the algorithm used for evaluation:

``` ruby Sortah core execution methods, #sort and #send_to. From lib/sortah/cleanroom.rb

    def sort
      catch(:finished_execution) { run!(@pointer) } until @pointer.is_a?(Destination) 
      self 
    end

    private
    
    def send_to(dest)
      @pointer = if dest.is_a? Hash
        Destination.dynamic(dest[:dynamic])
      else
        @__context__.routers[dest] || @__context__.destinations[dest]
      end
      throw :finished_execution
    end   

```

`@pointer` is either a Router object or a Destination object. #sort is
called with @pointer set to the root router, it calls a local helper `#run!` on
the pointer. `#run!` first calls the `#run_dependencies!` function on the
pointer, which runs all of the lenses (a noop if there aren't any), and then
does a `self.instance_eval` on the block contained in the router. This is what
ends up calling the `#send_to` method[1] which sets up the new pointer based on
the symbol it's provided (checking to see if it's a router first, destination
second, so a router will always override a destination[2]). `#send_to` then
makes use of the niftiest thing I hadn't known previous to this project. It uses
`throw` to unwind the call stack, which has the pleasant effect of burning up to
the nearest catch block (assuming it is a block which specifies the given symbol -- 
which is in `#sort`! The punchline is that now, `#send_to` acts like the `return` 
keyword. So we can do the nice:

``` ruby Isn't that pretty...
    sortah do
      router do
        send_to :foo if bar?
        send_to :baz unless quux? && 1 + 2 == 4
      end
    end
```

rather than having to spell out all of the details with a full if/else/end
block.

The point of all this is to show that delegating to a router with no lenses is
crazyfast, so the big price to pay is not routing delegation, but lens overhead.
This means that it will be better to minimize the number of lenses you need to
run, which means you should really try to minimize the number of lenses ran on
the root and "higher level" routers. Remember that routers form an arbitrary
graph -- you can happily call a router from itself if you like. There is no
stack to worry about, so while I can't think of a reason it would be useful to
recursively call a router, it is possible. What I see as more likely is a sort
of "two stage" approach to writing your routers, the first being "figure out
what lenses will need to be run on this thing", the second being "pipe through
something which runs a bunch of lenses and does the real routing for this
'class' of mail".

## Temporary regrets  

The big one is lack of _direct_ support for maildir[3], at the moment you'll have
to tack on `new` manually. I don't see much point in preserving mbox, I don't
like the format. I don't even really like maildir -- but I haven't thought of
something better yet. Future non-sortah plans of mine include a mail reader (I'm
shopping for names at the moment, if anyone has a brilliant idea), so perhaps
that will become a labratory for new mailbox format experiments. I'm not
particularly opposed to maildir philosophically, it's just a bit of a pain to
get it to work with, for instance, mutt. If I had a "Longterm regrets" column,
it would be "Thinking that it would be cool to use mutt" -- seriously, I don't
know how people deal with the miserly state of the thing, it's got that "I'm a
unix tool for unix nerds and thus I must be configured only in this
incomprehensible language which tries to merge bash and vimscript and shit all
together." 

I _like_ bash, I _like_ unix, I _am_ a unix nerd, but this shit is unacceptably
awful. 

New rule, if you use single-character symbols as a "shortcut" for anything that
is actually a _variable_ in your script because _you can't be bothered to do
proper substitution of shit_. You are to be paraded through town, naked, with a
feather shoved...

Well, you get the idea, I don't like muttrc.

Getmail is kindof crappy too, but I'm less hateful of it because it has the
fringe benefit of being easy to tweak from a cut-and-paste I stole from someone
else.

I'd really like to roll in getmail in some way to sortah. I know that the unix
philosophy says "one thing, one thing well", but it always felt to me like
retrieval and processing were really the same thing -- acquisition. In my head,
my mail is _already_ in the sorted format, in theory, I should just be doing a
glorified `scp`. So naturally I dislike a tool which messes with that mental
model. Similarly reading and sending email always struck me as a "delivery"
problem, though the model there is perhaps less clear. 

There are a few other minor things I wish were different in sortah, most of them
are listed in the "FUTURE_PLANS" doc in the repo, but largely I'm looking
forward to getting my own sortah system set up, getting it working (if
temporarily) with mutt, and getting off the GUI with my email as it is.

--------------------------------------------------------------------------------

[1] don't mind that it's private, it probably shouldn't be since it's getting
called externally, but `instance_eval` bypasses normal checks, and the
important thing is that you shouldn't be able to call it from any context but a
`sortah` block. 


[2] this is on the list of, "Things which are a bug no matter which side you
choose" -- in this case, I chose router because it will cause your filter to
hang, rather than gleefully put your email in the wrong (or at least unintended)
spot. I figure it's better to be loud and wrong than quiet and wrong...

[3] This, as of publish time (12-16-2011) is no longer an issue, sort of. Sortah
has really hacky support for maildir. It's not been an issue for me in practice
yet, but I have more pressing issues in terms of email to deal with (eg, I hate
every textmode email client I've tried, so I'm writing my own).
