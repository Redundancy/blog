+++
date = "2016-06-07T14:46:39Z"
title = "Plan for Change"
description = "Planning for long-lived services"

tags = ["services","maintenance"]
categories = ["services", "maintenance", "engineering"]
+++

# Change
Software can change a lot over its lifetime because technology is moving faster, and I believe that software is living longer when operated as services. Change, driven by evolution and revolution in competitive technologies, huge changes in platforms, business models and shifts in customer expectations require us to be agile.

I don’t mean that we have to adopt Scrum, but that we need to fundamentally consider software development with an agile viewpoint that accepts change as inevitable — you will be disrupted every 5 years or so, and you’d better be able to adapt. Maybe it’s new hardware, the cloud, some new database solution, VR, mobile, micro-transactions, free to play, digital distribution, services, new middleware…

## Change is hard
Scalability challenges when launching a new product and experiencing success, are foreseeable, if unfortunately not givens (unless you’re that person who makes up the totally unrealistic sales forecasts).

Other things, you don’t see coming — they come out of the blue, from a garage or from a press conference. They change your business, like it or not, and to compete you’ll have to adjust on a timeline set by your competitors and new startups that lack your technical debt.
Some change is easy, while other changes could require you to rebuild everything that a particular technology, architectural choice or set of assumptions touch.

## The cost of change
The breakdown of cost between building code and maintaining code is a very interesting, and probably not well enough known concept. Various sources have put the cost of new code at around 20%, with maintenance costing 60 to 80% of the total.

Some impressions:

1. It’s possible for developers to build more code than they can maintain
2. One “bad” developer creating new code at full steam could cost you 4 full time developers
3. If you can find a way to halve your lifetime maintenance costs, you could almost triple your development costs and still break even
4. If you halve your team after 3 years of development when you ship, you could have 24 years of maintenance work for your remaining team already
5. If done in small enough chunks, rewriting code rather than maintaining bad code might not be so crazy

* http://www.irahul.com/2005/10/software-development-vs-software.html
* http://galorath.com/wp-content/uploads/2014/08/software_total_ownership_costs-development_is_only_job_one.pdf
* https://en.wikipedia.org/wiki/Software_maintenance
* https://en.wikipedia.org/wiki/Software_evolution

Note that we should use these figures as educational warning, not Truth.
## When we pay
At the start of the software life-cycle (the greenfield stage) most of our effort is in new code and we feel like we’re moving incredibly fast. At later stages, there’s every chance that we really are moving much more slowly.

There is never an ideal time to pay down the cost of code that’s hard to maintain.
During our growth phase, we’re always trying to finish and ship the product and more features, and once we’re over the hump we’re almost always fighting to change things to get customers back against a system that resists it.

## Reducing the costs
The first step is understanding that systems that we create are an ongoing burden and liability — their maintenance will cost us, and the opportunity cost will prevent us from creating new systems to create business value.

The next step is actively developing in a way that reduces the costs as much as you can.

### Don’t maintain it yourself
If a system is not providing you with a competitive advantage, consider if you can avoid or share the costs of maintenance. Licensed software or support contracts can offset what you need internally, and the cost is usually shared with other customers (but so are the prioritization of improvements or features).

As a side note — taking someone else’s code and modifying it for yourself, without submitting it back, is massively painful, as you will now need to integrate changes from the mainline to stay up to date.

### Don’t pay the cost
I include this “option” only based on how often it’s chosen without anyone actually talking about it.
This is when you put up fences around systems or code, and tell each other “here be dragons”. You actively avoid doing changes or fixes that might require you to touch the code, or occasionally run in there to do the smallest changes you can and hope it didn’t break everything. You might even put it on a prioritization list, clear in the shared understanding that it’s never going to be worth going there.

There are some conditions where this approach might pay off — particularly when you’re banking on being able to throw the system or code away before you need to make any bigger changes. This is a form of gambling though, and as time goes on (particularly as knowledge of the system decreases), if you do eventually need to change it you could be in a lot of trouble.

### Don’t change, rewrite
In some situations, once you know your requirements better as well as the problems, you’ll be in a significantly better position to create systems that handle them without too much speculative generality. This approach can buy you a complete rethink of your dependencies and tech stack.

The key is to be in a situation where a rewrite is a practical option, and this is typically where the size of the rewrite is small and there are good options for ensuring compatibility with existing integrations. Note that this is at least some of the logic around Micro-services.

### Isolate change
Assumptions, tight coupling, dependencies and leaky abstractions all tend to work to increase the amount of code that you need to change, and spread it around your systems.

Good abstractions, clean APIs, SOLID and firebreaks like DLLs, services or other processes can help to constrain where change is required, at the cost of up front effort in design and versioning.

Choices that affect your entire stack, and particularly frameworks, can be particularly dangerous in terms of requiring large (possibly unfeasible) amounts of work to change.


### Refactor continuously
Making changes to code that don’t result in more features can freak out producers and your QA (maybe even Ops). Unfortunately, that probably means that they don’t trust you to make changes without causing everyone else a huge number of problems. As an engineer, or engineering team, it’s up to you to regain that trust by working out what practices you can adopt that increase the perception (and hopefully reality) of the reliability of the work that you do and the overhead it causes on others.

Another approach is the Boy Scout Rule — whenever you have to change code, try and make sure that you leave it in a better, not worse, state. Since there’s a commitment to sort through the result of the change anyway, the overhead of small refactorings made in the same context is significantly reduced.

### Make change easier
You can write code that’s written to be maintained by making it easier to read and reason about. Make your names clearer, your methods shorter, your state changes easier and optimize it for the reader before the machine. Don’t try and show how clever you are with language features and syntax.

Write code as if you, 5 years younger and in a hurry, will be the next person who has to deal with it.

### Change with confidence
A big part of changing code is knowing when your changes work, and whatever other requirements of the original code remain are all met.

Good automated tests can give you a lot of confidence, while bad automated tests will get in the way of your changes. Frequently, it comes down to what you’re changing, and what your tests are trying to ensure stays true — if your tests are working on the detailed internals of your system, they will discourage change. If your tests work at the same level that your customers work, then changes to the details are up to you. Tests of the details are still ok, as long as they happen at the same level as that detail, and can be easily removed if that part of the implementation is no longer relevant.

Documentation of high level goals (or purpose), logical edge cases and other non-functional requirements (such as performance) are also very useful for anyone attempting to ensure that their changes will cover everything that’s required.

There are also plenty of good ideas for running old and new solutions side by side for comparison - this can allow you to compare behaviour and output, with choices about how much it may affect your production environment. See: Canary Deployment, Shadow Deployment, GitHub’s Scientist library.

# Some Good References
* http://www.amazon.com/Leading-Transformation-Applying-DevOps-Principles/dp/1942788010
* http://www.amazon.com/Working-Effectively-Legacy-Michael-Feathers/dp/0131177052
* http://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882
