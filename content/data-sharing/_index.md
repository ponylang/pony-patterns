---
title: "Overview"
section: "Data Sharing"
layout: overview
menu:
  toc:
    identifier: "data-sharing-overview"
    parent: "data-sharing"
    weight: 1
---

"How can I send my mutable `ref` value from one actor to another?" It's a question that new Pony programmers often ask. The short answer is, you can't.

Sharing mutable data between actors is fundamentally unsafe. The Pony compiler will prevent you from doing it. However, that doesn't mean that you can't accomplish your goal.

In this section, we'll cover some patterns for sharing data between actors including a couple ways to you can "share" `refs` between actors.
