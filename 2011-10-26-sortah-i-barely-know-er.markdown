---
layout: post
title: "Sortah? I barely know 'er!"
date: 2011-10-26 08:04
comments: true
categories: [open-source, code-I-wrote, email, email-sorting]
---

## What is Sortah?

Sortah is a embedded DSL for Ruby, which provides a framework for writing email
processing routines in Plain Old Ruby Code (PORC[1]). It looks a bit like this:

``` ruby sortah example http://recurs.es/2011/10/26/sortah-i-barely-know-er/ original article
    sortah do
      maildir "/home/jfredett/.mail/"

      destination :work, 'work/'
      destination :gmail, 'gmail/'
      destination :personal, :gmail           # aliases 'gmail' destination
      destination :trash, :abs => '/dev/null' # an absolute path to somewhere on
                                              # the system


      lens :spam do
        blacklist = CSV.read("blacklist.csv")
        email.body.split.inject(0) do |a, word|
          a += 1 if blacklist.include? word
        end
        true if a > 20
      end

      router do
        send_to :work if email.to =~ 'joe@work.com'
        send_to :spam_filter
      end

      router :spam_filter, :lenses => [:spam] do
        send_to :trash if email.spam
        send_to :personal         
      end
    end
```


The above is a pretty straightforward example of how sortah handles email.
Things to notice include how 'lenses' -- which are like filters your email will
pass through, which may 'color' the email with additional data -- are specified
as dependencies of 'routers' which provide a way to specify routing logic. The
main idea is separation of email _processing_ from email _routing_. The former
being tasks like running an email through a bayseian filter, or inspecting for
presence in a blacklist. The latter being tasks which use that information to
determine where things go on the system. Sortah is smart about how it runs
lens-dependencies, so that they only ever get run once (a bit like rake task
dependencies), this allows you to build up a library of lenses which you can
keep around even when you need to change sorting logic, and vice versa --
encaspulation, it works!

## Enough about what it is, why is it?

I wrote sortah because things like procmail and seive -- while (at least the
former) is venerable and still very powerful -- have utterly arcane syntax, and
often make even the simplest of processing tasks very difficult. For instance,
look at this seive example:

    # Sieve filter

    # Declare the extensions used by this script.
    #
    require ["fileinto", "reject"];

    # Messages bigger than 100K will be rejected with an error message
    #
    if size :over 100K {
      reject "I'm sorry, I do not accept mail over 100kb in size. 
      Please upload larger files to a server and send me a link.
      Thanks.";
    }

    # Mails from a mailing list will be put into the folder "mailinglist" 
    #
    elsif address :is ["From", "To"] "mailinglist@blafasel.invalid" {
      fileinto "INBOX.mailinglist";
    }

    # Spam Rule: Message does not contain my address in To, CC or Bcc
    # header, or subject is something with "money" or "Viagra".
    #
    elsif anyof (not address :all :contains ["To", "Cc", "Bcc"] "me@blafasel.invalid", 
                 header :matches "Subject" ["*money*","*Viagra*"]) {
        fileinto "INBOX.spam";
    }

This example is similar in complexity to the sortah example above, and while it
is miles ahead syntactually, it's also ugly as sin. I wrote sortah because I
couldn't understand why anyone thought this:

    :0:   # Deliver to a file, let Procmail figure out how to lock it
    * ^From scooby
    scooby

    :0    # Forwarding; no locking required
    * ^TO dogbert
    ! bofh@dilbert.com

    :0:snoopy.lock  # Explicitly name a file to use as a lock
    * ^Subject:.*snoopy.*
    | $HOME/bin/woodstock-enhancer.pl >>snoopy.mbox

was a good idea. It's, frankly, impossible for me to wrap my brain around this
language -- seive is better, but it has the unfortunate design choice of mixing
business (routing) with pleasure (processing) -- I think sortah excels here.

One of the most fundamental things I thought should be available to the intrepid
email sorting loon is the full weight of a _real live programming language_.
Sortah provides one, Ruby. You can happily write a lens which provides you with a
piece of email metadata in the form of a custom class -- I actually outline such
a case in the README on [github](http://github.com/jfredett/sortah), where I use
a `Contact` class to help sort email to appropriate folders in a contact list.

Further, Sortah is declarative, which makes it _much_ easier to compose than
procmail or seive. In fact, in generally boils down to calling `load` somewhere,
and routing into the library you're loading. This means it's _much_ easier to
share common code. Rather than needing to manually bind to SpamAssassin, we
could bind to it once, and share, or -- even better -- make it available
directly in ruby.

## Alright, what's the catch.

Well, it's a bit slow at the moment, mosty because each email loads your sortahrc
every time it's called. A future release will fix that by letting sortah cache a
few emails, or maybe run as a server, or -- well I haven't figured out how I
want to solve that problem yet. It also doesn't provide a way to "re-sort" an
inbox, though this should actually be pretty easy to do with `find` and some
ingenuity. 

It also doesn't natively support maildir or mbox or anything, it just writes
files to the filesystem. You should be able to make it work with anything you
like in fairly short order, but it does take a bit of work at the moment.

Finally, this assumes you're using getmail, it assumes you send mail from
getmail to sortah as a `MDA_external`. I haven't tested other uses of it, so
YMMV if you try something novel (if you do, tell me about it!)

## Where can I get it?

You can get it from my [github](http://github.com/jfredett/sortah) if you want
the source, or you can do

    gem install sortah

in a gemset somewhere and use rvm exec, etc. There's a tutorial for setting it
up in the [github](http://github.com/jfredett/sortah) repo, the setup worked
for me, but it's not particularly optimized or highly tested. 

I hope you find sortah useful, I've had a lot of fun building it. I really hope
if finds some use amongst you few who found that mutt + procmail|sieve + getmail + 
blahblahemailtoolsblah was just too much work to get delicious commandline
email management find this to be a nice weight off your shoulders. I know it was
one of the things holding me up.

--------------------------------------------------------------------------------

[1] I'm making this a thing.
