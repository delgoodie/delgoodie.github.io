---
title: "Video Game AI"
date: 2024-03-18
description: "the stuff"
summary: "and how it differs from subconsciousness"
draft: true
---

My solution after listening to [Structure vs Style](https://www.gdcvault.com/play/348/Structure-VS):
### Structure:
There are stimuli which must fall into one of five categories (denoted by the verb, indicating the sensing type):
- Sight
- Hearing
- Vision
- Feel
- Internal
The internal is just the result of any internal state changes, eg the player gets too tired (although this could potentially be grouped with feel.
An example of a stimulus would be:

> When the player hears a dog bark

There are also nouns in this stimulus declaration: the player, dog, and bark
Nouns are “objects” with properties: player location, dog mood, etc
Stimuli can have clauses:
When the player hears the dog bark and they are in a bad mood
Here the clause is:
they are in a bad mood
These are “conditionals”
Finally there is the behavior:
shoot the dog.
So this follows the behaviorism idea of inputs and outputs, stimulus and behavior.
All together:
When the player hears the dog bark and they are in a bad mood, shoot the dog.

Behaviors can utilize nouns, here the dog isn’t any dog, its the dog that the player heard barking in the stimulus.
So this does sort of feel like a style vs structure. Obviously the implementation of the stimuli and behaviors is still in code but I think that is unavoidable. The main two problems I see are intuition at complexity and blending:
- Can this handle very complex and numerous behaviors with competing clauses etc. Is it intuitive what decision will be made?
- How can I easily blend / transform a specific “personality”?

To maybe solve blending, we can use roles which are a higher abstraction for behaviors. 
A role would be a collection of behaviors the fit a group

Soldier: a group of behaviors responding to military type stimuli (gunshots, enemies, etc)
Dad: a group of behaviors responding to child parent dynamics

Then these roles can be combined together to create higher level behavior designing.
(What if roles compete for stimuli / behavior? what if a role is almost good but fails in one case?)
Maybe we can also introduce parameters which have weighted effect on behavior (and thus role) expression (eg a shy soldier doesn't engage as much)
To solve the first problem is tricky, a really really good debugger would help but this requires testing cases which is far from ideal.
Intuition of complex personality might not be possible, after all, can we predict our own behavior in a hypothetical situation we’ve never been in before?

Use Psycology’s Behaviorism 
Here's a completely different idea:
Define behavior based on objects, not actors:

Rock: can be kicked, throw, held, etc
Oven: opened, set, closed, etc

Then actor can pick from available behaviors based on “personality”