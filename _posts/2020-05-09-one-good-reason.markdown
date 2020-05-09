---
layout: post
title: One Good Reason ..
date: 2020-02-10 18:00:00
description: To not use catch-all exceptions.
tags: [python]

---

I was listening to latest [Python Bytes](https://pythonbytes.fm/) podcast and one of the topics was about [warnings and error handling](https://lerner.co.il/2020/04/27/working-with-warnings-in-python/). Post starts with showing few examples of exception handling and then goes on to talk about  Python's [warnings](https://docs.python.org/3/library/warnings.html) module.



### And the light starts to blink.

Many documents and even static analizers will typically warn you about "too broad exception" - typically when you use a catch-all exception like:

```python
try:
  msg = "Hello World!"
  if "." in msg:
    print(msg)
  else:
    raise RuntimeException("Not Enough Cowbell!")
except Exception:
  ## cleanup handling here 
```

Why would this sort of "catch-all" exception handling then be considered as bad practice?

Lets consider that the `if`block in above piece of code represents our *business logic* and we know that it can throw some *set* of exceptions (besides the explicit `RuntimeException`). We could be lulled into false sense that if any error happens, we just simple cleanup and be done with it. Our code works as it should, right ? 

Let's make a small change from:

```python
  msg = "Hello World!"
```

to

```python
  message = "Hello World!"
```

We run the *business logic*, exception is thrown and it was handled up as previously. All is good, right?

Of course not! There is now a new exception added to the list that we are covering with our `except` that we where not expecting ;) 

And in this case, obviously you will spot in your editor or IDE that if clause uses undefined variable so thats easy to spot before even running the code but here's the catch. You could still have valid code but input's to your *business logic* could still generate exceptions. This broad exception handling will essentially hide issues in your actual code. It could be still be valid syntax and still do that depending on what your business logic looks like. And still,  your error handling does what it is supposed to but you wont see the error that you really should see.

Now that im thinking of it, I've actually been bitten by this sort of approach before.

Write a bit of throw away code while thinking to myself: I'll had naive error handling first and then add the proper stuff later. Then, introduce a logic issue that generates exception, try to run outer scope with some testdata and think to myself, what's wrong  with my data as it always fails ? Yeah, because my error handler was catching absolutely everything and I was thinking the failure was somewhere else!

