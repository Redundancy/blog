+++
date = "2016-04-02T14:46:39Z"
title = "File Static Data"
description = "Versioning game data in the VCS effectively, even for large amounts of data"

tags = ["FSD", "python", "data"]
categories = ["services", "maintenance"]
+++
Eve Online started out life early on with the “Database”. Because the DB existed already and was relatively convenient, we put data into it. Later, we realized that a lot of data was stuff that our game designers authored (like solar systems and their contents) that we called “static data”, and a clever system was created for versioning this data with revisions at a row level.
It didn’t seem unreasonable at the time, and the waterfall mechanics of our release process matched pretty closely with the mechanisms in the database versioning system.

Games can have multi-gigabytes of this “static” data. As the fidelity of our game worlds increase, the number of textures and geometry increases, and so does the density of information of “stuff” that the game typically needs to know — if our static data is in the database, an “empty” database populated only with the information about the world can start to get pretty big.

The main problem is that it’s very difficult to keep two different version control systems in perfect sync if they’re tightly coupled, and when you build your own system you’ll have a terrible time replicating all the features of a real VCS like branching. This became more and more relevant as the branching model of the game evolved.

Our branching model evolved because our needs evolved — as we moved to a live environment and became a larger team, we needed concepts like hot-fixes where a strictly ordered downstream flow of changes caused problems. We also needed feature branches, so that large features (and their data changes) could be developed for different releases simultaneously without blocking each other. None of these things fit with the supported model very well.

![](/fsd/eve-branching-model.png "The Eve Online branching model")
https://www.perforce.com/resources/presentations/merge-2013/continuous-delivery/versioning-everything-perforce

In 2011, after facing some incredible challenges stabilizing Incarna against work from multiple teams changing live database schemas and contents alongside code and assets at a rapid rate, we started a project to move away from the system we had used for ~6 years. In our retrospectives, every one of the teams in Reykjavik had highlighted the system (Branched Static Data) as a major impediment.

## Just Stop
There is really no value added to your company in building something to replicate the features of tools that you already have and pay for, and you already have a VCS. You also know that your source control provides, or can mirror any code branching that you have.

Our ballooning data requirements (unless you lean on procedural systems) force us more and more to transform our data into engine-efficient structures with careful management of loading, compatibility, memory, storage and other concerns.

It’s also important to realize that the needs of data formats for runtime, and the needs of data formats for editing are at odds. It requires a transformation step between the two not to compromise one or the other.

## Where
Once you adopt version control for your data, you need to figure out where you want data to live: this is a subtly difficult question depending on if you have a monolithic branch, or potentially multiple branches.

My recommendation is that you should always try to ensure that your source data defines something in one place (for consistency) and that locality should be based on ownership and/or coupling, and that system is then authoritative.

If someone depends on data that’s owned by something else, especially versioned or deployed at a different rate, you really need to think about how it can get that data to stay in sync, how it can be delivered to decouple the underlying format and implementation and how you manage that consistency problem in your runtime or deployment pipeline. In a world of high availability micro-services, this becomes all the more relevant.

## How
Your next challenge will be your source format: while relational databases are great, their normalized and row-based structure means that they suffer from extreme dissonance with expressing features - adding a new entity to a game could involve data in many tables. Modifying a single source file per table would mean a lot of merge conflicts, and those conflicts are going to be difficult to resolve by a person because of it.

There is a solution to this involving storing the log of changes (either to tables, or objects), rather than the data itself — merging branches involves replaying both modification logs and looking for conflicting actions. This should work, but it isn’t my favourite approach as it still doesn’t solve the human element of merge conflicts. (See: event sourcing)
To do this, we must consider how it works for the data that we, as engineers are used to merging as part of using these tools, and how we could make that work for others.

When we work with code, we typically have rules about file sizes to break things up into logically connected areas of code in files and packages. In an ideal situation, when we make a change, we can see most of the context of that change in a single view, such as in a class, or package. Changes that involve touching every file in source control are a misery to review, and may hint at far too extensive coupling across the codebase.
> Context is critical for merging text, as is human readability.

## Designed for merging
This illuminates an approach to dealing with our data — like with NoSQL and structured / document based data, we want to capture a high level context that matches how we edit the data. If we work on the level of entity types, we’d like to try and capture all of the pertinent information about an entity in a single file, and then conflicts in this file reveal a conflict in a related change.

To make the best use of existing tools, we want a format for expressing this which best works with text. We want to strip as many numerical flags as we can into something that people can look at and validate without cross referencing, and we want a format that’s well structured for line based-merges.
For us, the format was YAML — it allows comments (if needed), the lack of things like opening and closing braces help, as do the lack of odd rules on trailing commas (like in JSON objects) and it’s more readable and quickly understandable than XML, as well as being generally easier to work with.

To this, we added some modifications and requirements:

* Enforced ordering (key-sorted) when marshalling maps and object to prevent attributes moving and creating unnecessary diffs.
* Games are frequently very dependent on floating point precision. We created a library that overrides the standard YAML formatting to maintain bit-for-bit floating point accuracy. (see references)
* Vectors are very common in our data. We specialize the case of 2,3 and 4-tuple floating point numbers to display them on a single line.
* We only allow strongly ordered sets instead of general lists. Merging lists without a clear ordering is very difficult to get right or find the right solution to automatically. This can be a burden on the data design, but it helps with tools for automatically resolving conflicts. To do this, we
* Prefer GUIDs / UUIDs for object ids. This avoids the need for a globally incrementing ID system if you can. However, for compatibility we enabled a very simple REST based service for creating unique incrementing integer IDs.

Just like with code, we should to attempt to limit each file to a reasonable size when possible to reduce the burden on the merger.
Although the file based results are merge-able, adding the values 2 and 3 to a map containing only 1 will in both cases result in lines at the end. We built some automated tools that take advantage of knowledge of the structured nature of the data like this to be able to provide automatic 3-way merging for many common cases, integrated into P4Win / P4V extensions based on file extension.
### Not Unstructured
An important part of this is that the data we save is not unstructured — since this is user-created and edited data we save ourselves a significant amount of pain by doing as much validation on the data as possible, and keeping it well structured.

We created a library to express this, after finding JSON Schema limited in its ability to cleanly express objects, and a number of its library implementations poor at communicating the location of errors. Based on the schema we created another system for automatically building indexed files with as little work as possible, to ease adoption in simple cases and minimize memory usage.
### Directory Structure
We’re very likely to end up with a large number of files. Windows, certainly, doesn’t deal with thousands of files in a single directory, and it’s not great for users.
Since we have a hierarchical representation (the file system) to use for this, we should try and make as good use of it as possible. Our uses are for grouping related data and to potentially express information in a parent directory that applies to those underneath.

We should attempt to use the structure to express as much information as clearly as possible to our users, who are the people creating, editing and merging data, while keeping data unique. We should also attempt to ensure that deleting a directory is (for the most part) enough to remove the data.

## To Build
By now (at least for some projects), we have a large amount of data, potentially expressed in a file structure hierarchy, containing large numbers of structured files that express concepts. The file system hierarchy is even important. At CCP, we have around 1.2 Gigabytes of data to describe solar systems and their contents in tens of thousands of files. It’s just not efficient to load at runtime.

If you’re tempted to go this far, you’re going to be staring down the gun of quite significant build times and complexities. A build process provides a layer of abstraction between this format and layout, and potentially multiple expressions of it (some based on relevance to client/server, others based on format like the database). Iterating over those files once might be expensive, doing it multiple times will be prohibitive.

The inspiration for solving this problem came from big data and NoSQL — we need to make this process parallel using a framework like MapReduce, running multiple mappers and reducers over the data while only loading (and parsing) it once.

The solution we built, FSD MapReduce uses “jobs” which are composed of:

* Glob-like file descriptions for the input files
* Glob-like descriptions of “directory files” that are available for the mapper as a stack
* A map function that may yield multiple results (or none) per mapped input file
* A reduce function
* The ability to sort based on the first result of the map function, and merge the sorted results of each thread/process efficiently into a fully sorted result

As an example of a job, imagine that we want a list of planets. The planets are contained in solar systems, and there may be many per solar system file. The mapper runs on each solar system file and spits out each planet ID and some other information. We want to know what solar system a planet is in, so we pull that information out of the file, but the solar systems are contained in a constellation, so we also pull that out of the hierarchy. Simultaneously, we’re also pulling out moons and stargate relationships in other jobs on the same file.

Files and directories not potentially matching are skipped entirely. This enabled us to build and stream large indexed files as well as CSVs appropriate to efficient bulk loading into the database where required. Instead of 20–40 minutes per job, we build the whole map project in 10 minutes, using all cores of a 24 core machine.

## Running Locally
At CCP we typically check in built data as part of a “sync and run” policy. This has the advantage that in Perforce, many users and systems can run without the source static data, and only some targets for the built data.

## Deploying to a database
We try to avoid the need to put data into the database these days, but sometimes you need it. Bulk Copy Program (BCP) and other optimized data loading solutions are your friends.

For each table we deploy we maintain a hash of the file that would fill it, and on deployment compare the hash of the current, and new data to the hash. For 6 million rows of asteroid data, this can save us a lot of time. We then ensure that any extra generated data is added by running table-based denormalization procedures.

## Delivered Data
Some data was initially shared by Dust and Eve by direct reference to the database in the Dust builds. When Dust was separated into its own depot to allow development on its own release cadence, we started building subsets of the data in a pre-agreed format which Eve would continue to deliver for branches regardless of where the data originated. This contract allowed Dust to develop build processes based on a maintained standard, not an internal representation subject to change.
> Do not let others create a support contract you never agreed to by reading your internal data directly.

## Modules and Standards

Our approach to solving this challenge involved two key principles:

* The solution elements should be modular
* We should use standard approaches where possible

We mostly chose standard editing formats to enable (careful) compatibility with libraries and tools, and built off features we had in source control and the file system.
Each of the solutions we built was as decoupled from others as possible — you could use schemas alone and/or the map reduce framework, compatibility ID generator, and structured object diff-merge tool. This was decided to ease the burden on future changes, if any one solution could be replaced with a better one, or requirements changed rather than forcing a one-size-fits-all solution.

## Integration Transformations
A downside of this approach is that if you transform the manner in which you store data inside a branch, you must account for merging changes from other branches into the data you transformed as your change moves through branches.
## Live Consistency
Once data hits a production database, we must consider the issue that any persisted user data (such as inventory items) that references static data can cause us a problem. This must be dealt with in one of the following manners:
## Don’t delete things
* Write code tolerant of the missing references
* Clean up the database by changing the references or removing them

Eve Online uses a combination of the first, and the last approaches, aided by database maintenance windows for deployments.
Keep in mind that there may also be other systems (analytics in particular) that may require information to remain available and relevant for historical purposes.

## Approach
At CCP, a significant aspect of the project was to deliver value quickly without huge interruptions or efforts for teams that had features in-flight. Concerns were based around moving the locations that code-data inconsistency could appear to code/data-data, which led to an approach of “peeling the onion” by starting with less-connected data rather than data referenced by key in the database.

Initial conversions were significantly more relational in design and data layout, since there wasn’t enough information to build a full context. Our first conversion was a single large YAML file containing an id->object map, which also served as a good way to prove the concept of it being merge-able. As we built more tools and learned more lessons, our later efforts improved.

We built tools to help ensure referential integrity with data that was still in the database, using some of the information from the database.
One example was where we had moved the information on the mapping of types to their graphical representations across. A significant use-case for designers was copying existing types, so we built a system that ensured the intent (that type Y had the same graphic as X) was honoured. For complex cases, manual resolution was required.

# Results
The results at CCP have been good — although changing something so (unfortunately) deeply coupled into our code-base was challenging, the project was taken on in a very incremental manner, with every effort to maximize compatibility.

Since its introduction in 2012, new teams have adopted it and applied it to new systems as well as continuing to port old systems (always a good sign). Where some teams previously had significant difficulty adding new information to static data, many now barely need to think about it.
Teams have also replaced the prototype editor that was built for it, which is an excellent sign of being invested and taking ownership, as well as adding new runtime formats.

In terms of effort, the largest investments were based around tooling and particularly maintaining compatibility and consistency with the old systems.

# References
A number of original influences and references were lost with the demise of AltDevBlogADay. I’d like to call out work published by the BitSquid team, and Jonathan Blow. A lot of the work was going on simultaneously, but it certainly influenced our thinking.
### Event Sourcing
* http://martinfowler.com/eaaDev/EventSourcing.html
* http://codebetter.com/gregyoung/2010/02/20/why-use-event-sourcing/

### Floating Point Representation Precision
* https://randomascii.wordpress.com/2012/03/08/float-precisionfrom-zero-to-100-digits-2/
* http://the-witness.net/news/2011/12/engine-tech-concurrent-world-editing/

### Collaboration and Merging
* http://bitsquid.blogspot.is/2011/03/collaboration-and-merging.html

### Issues with real-time collaborative databases
* http://www.gamasutra.com/view/feature/3991/collaborative_game_editing.php?page=2
* http://www.gamasutra.com/php-bin/news_index.php?story=25316

### Branched Static Data
* http://talkminer.com/viewtalk.jsp?videoid=bliptv3258101&q=#.Vv_9XPmLTIU

### File Static Data
* http://www.develop-online.net/opinions/how-continuous-delivery-keeps-the-eve-universe-online/0214197
* https://www.perforce.com/resources/presentations/merge-2013/continuous-delivery/versioning-everything-perforce
