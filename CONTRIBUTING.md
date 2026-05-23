# Contributing to Pony Patterns

Hi there! Thanks for your interest in contributing to Pony Patterns. The book is being developed in Markdown. We welcome external contributors. In fact, we encourage them.

Please note, that by submitting any content to the Pony Patterns book you are agreeing that it can be licensed under our [license](LICENSE.md). Furthermore, you are testifying that you own the copyright to the submitted content and indemnify Pony Patterns from any copyright claim that might result from your not being the authorized copyright holder.

## AI-assisted contributions

We appreciate contributions, whatever the source. If AI tools help you contribute to Pony, we're glad to have you. Many of us use them too.

That also means we know what AI-assisted writing looks like when no one's reviewed it, and we won't merge PRs that read that way. If your contribution is clearly AI-assisted, you have an extra responsibility before opening it.

The Pony project publishes a set of LLM skills at [ponylang/llm-skills](https://github.com/ponylang/llm-skills). Familiarize yourself with what's there and make sure your contribution uses the ones that apply to it.

One skill isn't optional: `pony-docs-review`. We recommend running it against any contribution. For clearly AI-assisted PRs, we require it. Run it. Understand every finding. Then address it — by fixing it, or by dismissing it with a justification you can defend in the PR. The review surfaces the kind of issues a human maintainer would catch, but it isn't infallible. What you can't do is skip a finding because you didn't understand it. Don't open the PR until you've worked through them all.

If the review surfaces something you don't understand or don't know how to answer, stop. Don't open the PR. Come to the [Pony Zulip](https://ponylang.zulipchat.com/) and ask. We are happy to teach. We enjoy helping people learn Pony.

What we won't do is teach an AI. If a PR's open questions can only be resolved by maintainers explaining things to you so you can relay them to your AI for the next round, that's not a review. That's us teaching a tool that won't remember any of it. Our time is better spent on the contributor who came to Zulip first.

You don't have to disclose AI use, though saying so up front often makes review faster. What you do have to do is stand behind the work. If you can't answer questions about why your PR looks the way it does, it isn't ready.

### Junk PRs

This isn't new policy. Long before AI tools were any good, low-effort PRs that wasted maintainer time could get a contributor blocked. That hasn't changed. What's changed is how easy it has become to produce a junk PR. One won't get you blocked. A pattern of them will. How many makes a pattern is our call.

We don't owe anyone our time. We choose to give it. Don't waste it.

## How to format your Pattern

Pony Patterns is a series of cookbook style recipes. What does this mean? Each section of the book should be focused on solving a specific problem in Pony. Additionally, it should follow the pattern "Problem" -> "Solution" -> "Discussion".

"Problem" should be centered around what we are trying to accomplish. Give some background on why a person might want to read this pattern. Keep it reasonably short. Most people will find the pattern they need by scanning through the problem sections. They might also find it by searching so make sure to try and include like keywords in the natural flow of your description.

"Solution" should give code that solves the problem and then breaks it down into smaller chunks for discussion of what the individual section of code do. You don't have to cover every section of code, just the parts that are tricky or specific to the solution. Keep the code as minimal as possible and focused on the problem at hand.

"Discussion" should cover any additional content that might be of interest, what other solutions the reader might want to look at and any general conclusions to be drawn. Is the solution slow? Potentially hard to maintain? That should be covered as part of "Discussion".

Each pattern should start with the name of pattern as a level one header: `#` in Markdown. With "Problem", "Solution" and "Discussion" each as second level headers: `##`. If you need to have any subsections, make them a third level heading: `###`. If you find yourself reaching for a forth level heading, stop and figure out a different way to present the info in that section.

Avoid hard-wrapping lines within paragraphs (using line breaks in the middle of or between sentences to make lines shorter than a certain length). Instead, turn on soft-wrapping in your editor and expect the documentation renderer to let the text flow to the width of the container.

## Where to put your pattern

All patterns go in the top level of the repo within a directory for the appropriate section. If an appropriate section doesn't exist, create one and then add pattern within that directory. For example, to add a new section called `Networking`, you'd create a new folder `networking` then you can add your pattern to the directory. If you are creating a new section, add an `index.md` file that has the name of section as a level one entry: `#` in Markdown as well as description of the type of patterns to be found in the section.

## How to update the Table of Contents

Table of contents is handled by the `Summary.md` file. Each section of the book will appear as a top level item in the list contained in `Summary.md`. Each pattern in turn appears as sub item beneath that. If you added a new section, don't forget to link to both the section `index.md` as well as the content for your pattern.

## How to submit a pattern

Once your content is done, please open a pull request against this repo with your changes. Based on the state of your particular PR, a number of requests for change might be requested:

* Changes to the content
* Change to where the content appears in the Table of Contents
* Change to where the markdown file for the pattern is stored in the repo

Be sure to keep your PR to a single pattern. If you are working on multiple patterns, make sure they are each on their own branch and that before creating a new branch that you are on the main branch (others multiple patterns might end up in your pull request). Each PR should be for a single logical change. We request that you create a good commit messages as laid out in  ['How to Write a Git Commit Message'](http://chris.beams.io/posts/git-commit/).

If your PR is for a single logical change (which is should be) but spans multiple commits, we'll squash them into a single commit when we merge. In that case, please make sure that your first comment on the PR is the final commit message we should use.
