---
title: "My OOP Style"
date: 2019-04-17T20:40:00-05:00
---
I love Object Oriented Programming. This seems to be a somewhat controversial
statement these days, but I love how good OO code can model reality and feel
obvious to the human brain. To me, code that tells a good story might be one of
the most valuable properties of a system.

I spend a lot of time trying to make my code tell a good story. I love to find
a mental model that bridges human understanding with computer logic. Here are
some things I do in order to write that story.

## Spike, then cherry pick

First of all, I like to experiment. I rarely find the best narrative on my
first try, so I create a branch and move things around. I try thinking about
public interfaces that feel ergonomic and pretty, then run the test suite to
find out if such an interface is possible. I go with my gut and see where it
takes me, with the only hope that I get some insights about my story. Since
it's a spike, I'll throw this code away afterward. Sometimes I repeat this
process a few times until I'm comfortable with the story that I want to tell.

Once I know what's the goal of my refactor, I open a new clean branch and start
writing well-crafted commits with the code I'm most confident about. I do this
while testing my code thoroughly (which I don't do during the spike) and writing
good commit messages to explain my thought process. This feels like cherry
picking bits and pieces from the spikes.

## Find the objects

During the spike phase, I love using the [Reek
gem](https://github.com/troessner/reek) to try to find where each piece of
logic belongs. This often helps me identify new objects or rethink the identity
of some existing ones. Reek finds code smells, which then I try to
systematically fix.

Before I do that, there are two things I do to get the most out of Reek. First,
I limit the code I have in class-level methods. Many Reek checks don't apply to
those, so if I need to I create dummy classes so I can change class-level
methods into instance-level ones. Another thing I do is to remove guard clauses
and wrap the code below them into the `if` statement. The reason is that guard
clauses often make it more difficult to extract private methods, which is a key
refactor I use.

The first smell I look for is the [Too Many
Statements](https://github.com/troessner/reek/blob/master/docs/Too-Many-Statements.md)
one. This targets long methods, which I then split into private methods the
best I can. One thing I try to do is to keep the argument names match the caller
and the method signature, which will help Reek identify data clumps.

The next smell I look for is [Data
Clumps](https://github.com/troessner/reek/blob/master/docs/Data-Clump.md). This
will trigger when the same arguments are passed around through multiple
methods. When this happens, it means I need to create a new class to
encapsulate them. I don't worry too much about the name of this class yet
because I trust I'll be able to find a good one later when I have more
information.

Finally, I try to find [Feature
Envy](https://github.com/troessner/reek/blob/master/docs/Feature-Envy.md) and
[Utility
Functions](https://github.com/troessner/reek/blob/master/docs/Utility-Function.md).
These trigger when a method is using data that doesn't belong to it. The theory
here is that such code would be better if it lived with that data, so I try to
move those methods to the objects that it's using. When this is not possible
(maybe the objects are from external gems, or are Ruby native types...), I
create a new class for that code to live in. Once again, I don't worry too much
about the name yet.

## Class comments

Now I move on to the [Irresponsible
Module](https://github.com/troessner/reek/blob/master/docs/Irresponsible-Module.md)
smell, which will trigger on any module or class that doesn't have a comment. I
don't particularly love comments in code, but I find that trying to write one
short and concise comment about what a class does helps me find the name of the
class.

For every class, I try to complete the sentence that starts with "The purpose
of this class is to...". I try to avoid using the words "and" and "or" in such
a description. I also look at the data passed to the initializer (which defines
the identity of each object, what differentiates one instance from another) and
the public methods of the class (which defines how the instances are perceived
from the outside).

With all of those things in mind, I take a minute and try to write one or two
lines to describe the class, which should help me name it.

## Meaningful names

Once I understand what each class is supposed to do, I try to find a good name
for it. A good name should everything the class does, but not much more. This
means avoiding misleading names and too generic ones. This is more difficult
than it looks like.

One tool I use for inspiration is a [synonym
browser](https://www.thesaurus.com/). I use tools like these to try to hone
into the best word I can find for the situation. Maybe this also helps me
rename some of the attributes or the methods for the class. Finding good names
for classes is one of the most important things to do in order to tell a
coherent story with code.

## Polymorphism and factories

At this point, the code should be well named to tell a nice story. What I'm
looking for now is to untangle any parts that are hard to follow. One of these
things are classes where the same `if` conditional is repeated multiple times.
This is a nice opportunity to use polymorphism to reduce such repetition.

What I do is find where is my code creating instances of these classes and,
instead of calling the traditional `.new` method, I call the `for` method. I
then proceed to create such `for` class-method where I can run the repeated `if`
conditional once, then create a different object on each of the branches of the
conditional. One of the objects will be the same as the original one, the other
one will be made from a new class that I'm going to create as an exact copy of
the previous one. At this point, I can now remove the `if` from both of these
classes, leaving the appropriate branch on each one of them. This `for` method
is what's called a Factory Method.

It's possible the two classes now have a lot of repeated code. If so, I might
consider creating a class hierarchy out of those. This will depend a lot on the
context for the code that I'm writing since hierarchies are sometimes difficult
to get right.

Finally, I'll go through the naming process again to name this new class
appropriately.

## Final thoughts

This has been dense and it lacks examples all over the place. No worries! I'll
write more on this topic on further posts. If there are any parts you'd like me
to dive deeper into, please let me know in the comments and I'll make sure to do
it. Other than that, thank you for reading! I hope this helps you improve your
process while coding. I would also love to hear about your techniques as well!
