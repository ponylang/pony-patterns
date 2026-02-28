---
hide:
  - toc
---

# Streaming Patterns

Data rarely arrives all at once. When you're parsing structured messages from a network connection using `buffered.Reader`, you need to handle the case where a message is only partially available. Read too eagerly and you'll consume bytes you can't use yet, corrupting the buffer for the next attempt. The patterns in this chapter show how to work with `Reader` safely, ensuring you only consume data you're ready to process.
