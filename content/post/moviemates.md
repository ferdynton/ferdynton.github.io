---
title: "MovieMates"
date: 2018-08-04T20:34:33-05:00
---
This is the story of [MovieMates](https://moviemates.party), a service I built
with my roommates to help us determine what movies to watch.

I share an apartment with 3 other people and we love watching movies. Living in
a cold city like Chicago, it's one of our favorite hobbies during the long
winters.

About 2 years ago, we noticed that many times it would take us very long to
decide which movie to watch. We also realized that many times we would talk
about movies we wanted to watch but couldn't remember them whenever it
mattered. Sometimes when there were 2 people home to watch a movie, we couldn't
watch it because another roommate wanted too and we didn't want to watch it
without them!

Some of us at home are engineers and quickly thought of ways to fix these
problems. The first thing we thought of was having a Google Sheet shared among
us and adding rows with the movies each wanted to watch. Each of us would then
have one column with a dropdown for each row with three options to pick what we
thought of that movie: if you wanted to watch it, if you didn't care much, or
if you didn't want to watch it. With that information, we could apply filters
for the columns to figure out what movies we could watch depending on who was
available to watch it at that time. We called it MovieMates, here's what it
looked like:

![First version of MovieMates](/static/moviemates-spreadsheet.png)

Here's what it would look like if only Ainho and I wanted to watch a movie
without Fido nor Alex:

![MovieMates with filters applied](/static/moviemates-spreadsheet-filter.png)

This was a cheap way of satisfying most of our needs. It worked fine other than
being a bit annoying having to reset the filters constantly.

Soon, we started adding a lot of movies, the idea was a success! However, we
often had the problem of not knowing much about a movie. Sometimes the title
was even hard to find online. We agreed on making a new column for each movie
where we could manually add the link to Rotten Tomatoes to know more about it.
That way we could use the magical IMPORTXML function in Google Sheet to parse
the ratings from that URL. However, we started having issues and some ratings
started to not load properly:

![Rating import working and not
working](/static/moviemates-spreadsheet-ratings.png)

We knew we wanted more out of that spreadsheet and some of us were looking for
an idea to work on a toy project. We decided to start a Rails app to solve this
problem once for all. MovieMates as we know it today was born.

After some time developing MovieMates, it became ready to replace the
spreadsheet. I had it deployed on a Raspberry Pi locally and it did its job for
a long time as we kept improving the site very slowly.

Time passed, we replaced the spreadsheet with the app and we improved it over
time to fit our needs. We told other friends about it and turned out would
enjoy using it as well, so a few months ago we started working to get it ready
for the public. This meant enabling the site to work with different groups of
friends as well as making it prettier (when it was only for us, we cared more
about the function than the form).

Today [I announced the
project](https://twitter.com/Ferdy89/status/1025778772388274176) for anybody to
use. It's not the greatest piece of software in the world, but it's helped us
with our problem. I hope it helps you as well.

The site features group management for friends, searching movies on [The Movie
Database](https://www.themoviedb.org/) and importing its data, links to search
on popular streaming sites like Netflix, and provides an easy way of finding
out what movies are best to watch with your friends. We've come a long way from
that spreadsheet:

![MovieMates today](/static/moviemates-today.png)

Try it out yourself on [moviemates.party](https://moviemates.party) and please
[let me know](https://twitter.com/Ferdy89) what you think. I have plenty of
energy and ideas to improve it if it becomes a helpful tool for more people.

Thanks for reading and enjoy MovieMates!
