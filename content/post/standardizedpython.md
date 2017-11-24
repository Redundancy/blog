+++
date = "2016-04-23T14:46:39Z"
title = "Standardized Python installs"
description = "Using immutable Python installs, for great justice."

tags = ["python", "maintenance", "Programming"]
categories = ["services", "maintenance"]
+++

# Standardized Python installs
Virtualenv for Python is wonderful, and you should use it.

Our path to using it however, was rather rocky: working primarily on Windows, we were for quite some time victims of the Python community’s focus on Pip (and insistence on compiling everything during install).

Working on Windows, with a large number of non-technical users, the idea of trying to get your tools to compile on an artists’ machine should really scare you (never-mind the lack of testing / support and particular versions of visual studio that had to exist).

Our challenges were multiplied by the number of versions of Python that existed: Maya (a 3D tool) used a very old version of Python, and then threw the further wrench of using 64 bit versions. Our game engine used Python, but a custom version with updated compilers. Other tools had other requirements for various reasons, and we needed tests across the whole spectrum.

Initially, versions of Python was just put into source control, in what was called “shared tools”. These versions of Python were, to some extent, curated with updated packages and could be found on every machine, and they could be located relatively easily by batch scripts to get the right version of Python.

Of course, over time, these installations were added to with many packages, and used by tens of tools (most of which only a few people knew about). Updating our Python packages became a nightmare involving checking in a change and crossing your fingers that someone would notice and fix any dependency issues.

We realized the same thing as other Python users: you should manage and isolate your dependencies and environment per project, so that upgrades can be managed, tested and done individually with confidence about the scope of the change.

Just like picking up libraries randomly caused us significant issues, so did the pattern of just relying on python.exe being the correct version for our needs. It so often turned out that it wasn’t, and it caused us a lot of issues where we needed things to work on many different machines quickly and easily. CI machines could be especially problematic when they needed all the versions installed.

Our solution was to wrap the Windows Python installers, providing additional environment variables that were set for all users that would allow authors of tools to specify specific versions of Python. As an example %PYTHON_2_7_3_x86% would allow you to start with a specific python version.

>Explicit is better than implicit. — The Zen of Python

These base versions of Python were locked down so that no more could be installed into them, preventing people from installing and polluting them (therefore creating another shared environment) rather than using your own environment. This was an effort in trying to make it hard to follow the old anti-patterns.

Of course, you still need easy_install for the moment. Although the Python world has caught up on the utility of pre-compiled modules and created wheels as a standard means to support it, wheel support isn’t ubiquitous, and Windows is still somewhat overlooked as a platform for Python.

Far from trying to compete with virtualenv or providing something non-standard, we adopted it as much as we could, and simply provided some extra structure and convention to ensuring that you could get exactly the environment you wanted.
