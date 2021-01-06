---
 layout: post
 tags: database, best-practice
 title: How to Choose the Right Database for Your App
---
Making those database decisions a little easier and a little wiser

Working on new projects is always super exciting — we have the freedom to design and architect the things any way we want. But this planning, when not done properly, can cause us a lot of pain in the future.
Choosing your app database is one of the critical decisions you’ll have to make, and with this article, I intend to introduce you to various database options — along with listing some pros and cons to help you make wiser database decisions.

## Key–Value

Our database is structured just like a JSON object — every key is unique, and each key points to some value.

![Pic1](/images/db-pic1.png?style=centerme)

It keeps the data in the memory, which is very fast but comes with a capacity limit, so you can’t store a huge amount of data. And since there’s no disk involved, everything is blazing fast.
There are no queries or joins involved, so there isn’t much data modelling to worry about. Since there’s no schema, the developers always have the flexibility to change data as they may like.

![Pic2](/images/db-pic2.png)

When to use this technique
- This technique is mostly used as a caching mechanism for when some part of the data is being very frequently fetched and observed
- Hence, the key–value technique is widely used along with other databases as a caching mechanism

![Pic3](/images/db-pic3.png)
