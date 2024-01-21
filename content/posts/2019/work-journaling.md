---
title: "Work Journaling"
date: 2019-01-28
summary: "Overview of my work journaling system, circa 2019."
---

I started keeping a work journal as part of my work for [ASSET](http://asset-us.com/), back around June of 2018. Within about 2-3 days, I started another journal for my research work as part of [PASSLab](http://passlab.github.io/).

The motivation for journaling was basically a text-only extension of notes I was already taking with pen and paper. I found I could copy and paste command-line incantations, and could write down tricky settings for certain systems, so I wouldn't have trouble looking them up later.

Also, by tracking the rough amount of time I spent on tasks in my notes, it made reporting my hours much easier at the end of each month.

The key advantage of digital notes over paper notes is that I can do a text search over them to dig up old knowledge. Doing the same sort of thing with a paper-and-ink notebook is not nearly so easy!

The only thing I've missed from my paper notes so far has been the ability to stick pictures and annotations anywhere on the page. With digital notes (at least the way I'm doing them), I don't get that luxury.

## How I journal

Here's a rough snapshot of what a day's notes might look like:

```
------------------------------------------------------------------------------

# 2019-01-28

*Total Hours: 2.25 = 1.25 + 1.0*

 - 10:15 AM :: Starting work...
 - 10:42 AM :: Worked on the `foo` feature. Ran into issues with `bar` and `zeta`.
 - 11:04 AM :: Fixed `bar`. Turned out to be a unicode issue.
 - 11:10 AM :: Fixed `zeta` by putting in better validation before trying to calculate anything.
 - 11:20 AM :: Pushed up changes to Github.
 - 11:30 AM :: Stopping work.
 - 12:30 PM :: Resuming work...
 -  1:10 AM :: Worked on the `omega` feature. `omega` almost works, but it still fails on negative inputs.
 -  1:30 PM :: Stopping work.

## Zeta system

Lorem ipsum sincut dolor ....

```

I write my notes in Github-flavored Markdown syntax. Markdown is a text format I really like, because it looks a fair amount like what will be rendered when it's converted over to HTML.

Occasionally, I'll drop in links to stuff when it's relevant to an issue I'm debugging. This helps me find stuff later, and saves me from trawling through weeks of web browsing history to find a cool article or a post from a mailing list I saw somewhere.

## Format of the journal entries

My notes follow the format:

* Total Hours
* Time entries
* General notes

Because I keep to a set format, I was able to write a simple Python script that uses regexes to dump my hours for each month. I'm sure one can probably devise more clever methods, but it works well for me, and is pretty low-effort to maintain.

### Total Hours

This allows me to see how much time I'm spending each day, and helps me see how long I work in a session. The little math bit is mostly for error-checking.

If I'm working a contract job, sometimes I have to do some time-rounding, usually to the nearest 15 minute increment.

### Time Entries

The markdown list items have a set format of `- HH:MM AM/PM :: Notes go here...`, which makes them easy to parse, both visually, and with scripts.

I try to make a note of when I start and stop working, and that helps with hours tracking. Also, I try to make an entry whenever I'm making a big context switch, like after completing a feature, or solving a difficult bug.

### General Notes

Everything else goes in this section: code, design thoughts, limited TODO lists, etc.

I usually try to stick an informative section header over each distinct _thing_ going in this part of my notes (to make text-based search more efficient later).

***

## Environment customizations

Like most programmers, eventually, I felt the need to hack some extra tools and shortcuts into my workflow.

### Bash aliases for journals

In my `.bashrc`, I keep some aliases for quickly `cd`-ing down to the correct journal folder, and opening that journal in Vim. I've found that having the journals just one command away helps me get set up to work more quickly.

Typically, I've got 1 journal per workplace.

The bash alias lines look like:

```bash
# Work Journal quickstarts
alias readme-asset='cd ~/ASSET/work-notes-asset; vim README.md'
alias readme-passlab='cd ~/USC/PASSLab/work-notes-PASSLab; vim README.md'
```

Obviously, you should change those paths to what works for you if you try this approach!

### Timestamp shortcut in Vim

Eventually, I got tired of manually typing all of the little timestamps next to my journal notes. So, I created a Vim command binding to do the work for me! (Complete with correct formatting!)

I added the following to my `.vimrc`:

```vim
" Generate timestamps for worklogs, then go to insert mode at end of line.
nnoremap <leader>t "=' - ' . strftime("%l:%M %p") . ' :: '<C-M>PA
```

This command basically creates a new journal line, ready for me to type in some notes. Very handy, and reduces the time between having a thought, and writing it down.

## Closing thoughts

Work journaling has become an important part of my daily workflow, and helps me pick up context and ideas after setting them down for a bit. Over time, the journaling approach has saved me from a lot of wasted effort, mostly just by making old stuff (like process docs) easy to dig up again.

If you've never tried journaling during work before, give it a go! You'll be a little slower at first, until you get the hang of it. Over time, it hopefully will help you remember more, and make your life easier too.

Happy journaling!

