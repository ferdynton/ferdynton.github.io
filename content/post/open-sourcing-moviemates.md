---
title: "Open Sourcing Moviemates"
date: 2018-12-19T08:51:14+01:00
---
A few months ago [I talked about this project I've been working on]({{< ref
"moviemates.md" >}}) for the past few years,
[MovieMates](https://moviemates.party). Back in August, I made the service
public for anybody to use and a few days ago I also [open sourced its
code](https://gitlab.com/ferdynton/movie_mates). I wanted to talk a little bit
more about why I did this and why did I do it now.

The reason why I did this won't come as a surprise for those who know me. I'm a
big lover of open source (code in particular, but anything in general) as a way
of sharing progress and building as a society. I think building in the open not
only allows for more minds to join a creative process but also allows other
minds to build on top of whatever has been shared. Interestingly enough, I don't
think MovieMates will have many collaborators nor it'll be something for anybody
else to build on top of. I hope I'm wrong, though!

The fundamental reason why I open sourced MovieMates is that I've got no reason
to keep it private and because I'm proud of it. It's not a piece of art, but it
adds value to my roommates and me. There are other less important reasons I
wanted to open source it, like the small chance that somebody will join the
project, the chances it'll make users feel safer knowing what's going on with
their data on the site, and the possibility that I'll use it as a piece of code
to demo during interview processes.

There isn't anything revolutionary in this reasoning, so why did I choose to
open source it now after having been working on it for over 2 years now? The
fundamental reason is fear. Fear that I might have committed something I
shouldn't have (it's happened to me in the past and it wasn't pretty), fear that
I would get hacked (the original version of the site was deployed on a Raspberry
Pi on my home network), and mostly fear that my code wouldn't be good enough for
the world to look at.

In terms of fear of leaking credentials or exposing vulnerabilities, I now think
I was overly paranoid. There are [tools to check for any potential secrets
committed to a repo](https://github.com/awslabs/git-secrets), which MovieMates
has none. Also, if at this point in my career I'm capable of making a mistake
severe enough to allow for something like remote code execution while using a
mature framework like Rails, then I probably rather make such mistake on a toy
project rather than at work.

The most interesting fear I've had is the fear of my code not being good enough.
As if the world was going to come and laugh at my repository. Well, I now know
better: my code isn't, in fact, perfect, but probably no code is. I started this
project with the idea of finding the perfect Rails architecture and I struggled
a lot with that. Instead, I now focus on delivering features and keeping the
code decently understandable, that's it. I'm happy doing those things and I
don't mind much that my code is far from perfect. I hope some other people with
the same motivations come and give a hand!

So whether you're a developer and want to [give a hand with
code](https://gitlab.com/ferdynton/movie_mates) or if you're a movie fanatic and
want to [use MovieMates](https://moviemates.party) to build a collaborative list
of movies to watch with friends, come and say hi!

Thanks for reading!
