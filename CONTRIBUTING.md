# Contributing to Pony Patterns

Hi there! Thanks for your interest in contributing to Pony Patterns. The book is
being developed in Markdown and hosted at 
[Gitbook](https://www.gitbook.com/book/ponylang/pony-patterns/details). We
welcome external contributors. In fact, we encourage them.

Please note, that by submitting any content to the Pony Patterns book you are
agreeing that it can be licensed under our [liscense](LICENSE.md). Furthermore,
you are testifying that you own the copyright to the submitted content and
indemnify Pony Patterns from any copyright claim that might result from your not
being the authorized copyright holder.

## How to format your Pattern

Pony Patterns is a series of cookbook style recipes. What does this mean? Each
section of the book should be focused on solving a specific problem in Pony.
Additionally, it should follow the pattern "Problem" -> "Solution" ->
"Discussion". 

"Problem" should be centered around what we are trying to accomplish. Give some
background on why a person might want to read this pattern. Keep it reasonably
short. Most people will find the pattern they need by scanning through the
problem sections. They might also find it by searching so make sure to try and
include like keywords in the natural flow of your description.

"Solution" should give code that solves the problem and then breaks it down into
smaller chunks for discussion of what the individual section of code do. You
don't have to cover every section of code, just the parts that are tricky or
specific to the solution. Keep the code as minimal as possible and focused on
the problem at hand.

"Discussion" should cover any additional content that might be of interest, what
other solutions the reader might want to look at and any general conclusions to
be drawn. Is the solution slow? Potentially hard to maintain? That should be
covered as part of "Discussion".

Each pattern should start with the name of pattern as a level one header: `#` in
Markdown. With "Problem", "Solution" and "Discussion" each as second level
headers: `##`. If you need to have any subsections, make them a third level
heading: `###`. If you find yourself reaching for a forth level heading, stop
and figure out a different way to present the info in that section.

Please make sure to keep any individual line under 80 characters except in
instance where that would break link with the markdown (which only happens if
the linked text and url are more than 76 characters).

## Where to put your pattern

All patterns go in the `content` directory within an appropriate section. If an
appropriate section doesn't exist, create one and then add pattern within that
directory. For example, to add a new section called `Networking`, you'd create a
new folder `content/networking` then you can add your pattern to that directory.
If you are creating a new section, add an `index.md` file that has the name of
section as a level one entry: `#` in Markdown as well as description of the type
of patterns to be found in the section.

## How to update the Table of Contents

Table of contents is handled by the `Summary.md` file. Each section of the book
will appear as a top level item in the list contained in `Summary.md`. Each
pattern in turn appears as sub item beneath that. If you added a new section,
don't forget to link to both the section `index.md` as well as the content for
your pattern.

## How to submit a pattern

Once your content is done, please open a pull request against this repo with
your changes. Based on the state of your particular PR, a number of requests for
change might be requested:

* Changes to the content
* Change to where the content appears in the Table of Contents
* Change to where the markdown file for the pattern is stored in the repo

Be sure to keep your PR to a single pattern. If you are working on multiple
patterns, make sure they are each on their own branch and that before creating a
new branch that you are on the master branch (others multiple patterns might
end up in your pull request). Each PR should be for a single logical change. We
request that you create a good commit messages as laid out in 
['How to Write a Git Commit Message'](http://chris.beams.io/posts/git-commit/).

If your PR is for a single logical change (which is should be) but spans
multiple commits, we'll ask you to squash them into a single commit before we
merge. Steve Klabnik wrote a handy guide for that: 
[How to squash commits in a GitHub pull request](http://blog.steveklabnik.com/posts/2012-11-08-how-to-squash-commits-in-a-github-pull-request).


