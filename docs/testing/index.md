---
hide:
  - toc
---

# Testing Patterns

Most of us are familiar with unit testing code with more traditional concurrency models. In this case of threaded code, for many, this means throwing up your hands in disgust and walking away. Testing concurrency with threads is hard. Testing concurrency in Pony is much easier but if you haven't encountered anything like it before it can be hard to know where to start.

The patterns in this chapter will cover various ways of testing your Pony code. We will pay particular attention to how to test actors. Please note that this chapter assumes that you are familiar with the basics of using `PonyTest`, the Pony unit testing framework. If you aren't, please review the [`PonyTest` documentation](http://stdlib.ponylang.io/pony_test--index/).
