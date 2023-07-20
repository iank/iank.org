---
template: post
title: '"I PRESSED CAPS LOCK ONCE IN 1989 AND I NEVER LOOKED BACK"'
slug: i-pressed-caps-lock-once-in-1989
featured_image: 
draft: false
date: 2014-12-29T19:05:00.000Z
description: I get emotional about a Perl script
category: misc
tags:
  - robots
  - feelings
---
It's a widely reported phenomenon that the most popular or well-received thing a person creates is rarely the thing they are most proud of. Sometimes they grow to resent its overshadowing their more challenging or interesting output. Radiohead reportedly hates *Creep*.

[Ursula Vernon](http://www.ursulavernon.com/) is a Hugo award-winning writer and illustrator, but when I was a teen I went to a book signing to get her autograph on [this picture of a pear](http://ursulav.deviantart.com/art/The-Biting-Pear-of-Salamanca-29677500), which you'll recognize if you grew up on the internet at a certain time.

### My pear is an IRC robot called LOUDBOT

LOUDBOT was born at least five years ago. I was mourning the disappearance of an internet friend, who I'll call R. We'd spend entire days engaged in friendly but vitriolic arguments about his particular brand of stoner socialism or i-can't-even-remember-what, until he vanished, as internet people do.

Like "real" people you lose touch for a variety of reasons, benign or tragic, and often you never know. It's cliché but true to remember that there is aways attrition and that change can't and shouldn't be avoided. I've got to experience the relationships I have now and form new ones, while acknowledging the people who exist only in the past.

I processed R.'s disappearance in my own way by writing a robot to replace him. The bot went through my logs collecting every all-caps thing this guy had ever shouted at us, and when a user yelled it would yell back some mock-furious decontextualized incoherent epiphany or insult, like "IDEAS FLOW FREE WHEN TIME IS INCORRECT!"

### LOUDBOT got bigger than me

It was a simple Perl script, one hundred lines at most, and eventually I generalized it to repeat things that anyone had shouted in the presence of the robot. Anything shouted in all caps would be met with a random all-caps reply.

I started a [twitter account for the robot](https://twitter.com/LOUDBOT). Anyone on IRC could tell LOUDBOT to tweet the last thing it shouted. It [got into fights](loudbot_capscop.png) with other twitter bots[^1]. It [#engaged](loudbot_intel.png) with [#brands](loudbot_twc.png). While never particularly popular (it hovers at about 900 followers), it grew beyond my tiny corner of the internet. I had a surreal moment when I met someone in "real life" who followed LOUDBOT on twitter before she knew me.

The Perl script was written and re-written, overengineered, factored, and re-overengineered. It was a vehicle for me to play with NoSQL at the height of that movement (some versions were [web-scale](https://www.youtube.com/watch?v=b2F-DItXtZs)). It's integrated with my distributed swarm of self-managing IRC robots. Now it's over 1500 lines of asynchronous message-passing highly-available [OTP](http://www.erlang.org/doc/design_principles/users_guide.html) Erlang and Perl. I've forgotten everything there is to know about the code, but the nature of Erlang is that the thing has kept running for the last two years with no real maintenance or input from me. I call it "telecom-scale."

### Sometimes it's kind of the worst

Some of the quotes are stolen from other sources. There are political opinions and tasteless jokes. It is foul-mouthed and occasionally racist and awful. I'll delete the worst of it when I see it (as will others; anyone has this power), but the "loud database" is a product of the input of the users, and the users are [from the internet](http://www.penny-arcade.com/comic/2004/03/19).

But it sometimes reflects the creative humour of the nerds I enjoy spending time with, and the random responses are occasionally brilliant.

> < HighBit> THERE IS NOTHING MORE DANGEROUS THAN A HIGH-POWERED POLITICIAN ARMED WITH A TWITTER  
> < LOUDBOT_> THE INTERNET ISN’T PRIVATE? OH SHIT.  
  
> < HighBit> I ONLY EAT DRY FOOD FROM A BAG  
> < LOUDBOT> YOU HAVE THE POWER TO CHANGE THAT  

There is a small but devoted following. They spawned a mock religion. Some people insist it's sentient, maybe [Skynet](http://en.wikipedia.org/wiki/Skynet_%28Terminator%29), or that I have developed a secret algorithm for keeping LOUDBOT "on-topic". It's just a random select from a [SQLite](http://www.sqlite.org/) database.

> < LOUDBOT> IsharaComix: BACONIZED RESISTENCE  
> < IsharaComix> YOU SPELLED RESISTANCE WRONG  
> < LOUDBOT> I REQUIRE EPIC RUBY DRMAA  
> < IsharaComix> YOU SPELLED DRAMA WRONG  
> < LOUDBOT> EVERYTHING MUST BE IN ALL CAPS ALL THE TIME OR ELSE NO ONE WILL LOSTEN TO YOU  
> < IsharaComix> YOU SPELLED LISTEN WRONG GOD LOUDBOT YOURE BAD AT THIS  
> < LOUDBOT> THIS UNCHECKED AGRESSION WILL NOT STAND  

Over time, people grew accustomed to LOUDBOT's presence and the nature of their interactions with it changed. They didn't so much shout at each other anymore, they shouted for the robot to hear, record, and repeat later. The database grew to over 70,000 lines. This was different, and not what I had imagined it would become, but users and systems evolve as they interact with each other. That's neither right nor wrong, it's true and inevitable and interesting.

### Like the person it was initially created to replace, robots vanish too.

Our relationships with robots and systems, like people, are temporary. Trying to hang on to the way things were at the height of a friendship is impossible at best. It's been fun, but this system is stagnant, we've grown apart, and we're on to the getting-together-for-an-awkward-lunch-once-a-year phase of this friendship.

LOUDBOT isn't my [*Creep*](http://en.wikipedia.org/wiki/Creep_%28Radiohead_song%29), but it's time to shut it off. I'll always have this ridiculous [word cloud](loudbot_wordcloud.png). Thanks, Internet.

> -!- Nate has joined ##church-of-loudbot  
>  < Nate> wow it really exists...  
> -!- Nate was kicked by CAPSBOT [I CAN'T HEAR YOU, SOLDIER]  
>  < HighBit> CURRENT STATUS: GOING TO CHURCH  

[^1]: I'm told the creator of @CapsCop [secretly loves @LOUDBOT](http://twitter.com/natefanaro/status/7789580746)
