# Projects

## Open Source work at Styra

When I started work at Styra, I was assigned to help develop and maintain the Open Policy Agent project.
Even now, on the Enterprise OPA team, I get to do a fair amount of open source work for this project.

* [Open Policy Agent][gh-opa] :: [Docs][opa-docs] :: A powerful engine for running authorization policies written in the [Rego][opa-docs-rego] DSL. Can be used as a sidecar container, standalone server, or offline CLI program.


## Closed Source work at Styra

I currently work mostly on the Enterprise OPA project.
We frequently backport bugfixes and optimizations back upstream to OPA, which is nice!

* [Enterprise OPA][gh-eopa] :: [Docs][eopa-styra-docs] :: Styra's faster and more memory-efficient version of the Open Policy Agent. It supports connecting to a vast array of different data sources, allowing authorization policies to query databases and event streams, such as Kafka. I was brought in shortly after the project started, and have helped optimize, debug, add features, and more over its lifetime. I've also developed and helped maintain a good chunk of the CI and release engineering tooling the project uses.


## Earlier Open Source work

In no particular order:

* [mini-vm](https://github.com/philipaconrad/mini-vm) :: A register-based bytecode interpreter in about 300 lines of C code. Uses a jump table instead of the usual switch/case architecture found in most bytecode interpreters.
* [SOOMpy](https://bitbucket.org/sdii/soompy) :: A port of a civil engineering toolkit from MATLAB to modern Python. I started and maintained the early versions of the project, wrote most of the early documentation, and then set the engineers loose on the domain-specific parts. The original lab now maintains this project entirely.
* [omaha](https://github.com/philipaconrad/omaha) :: My senior year Capstone project—hacking on voting machines. We got most of the way to an information disclosure attack on the PEB cartridges used to store votes.
* [ompt-50-rs](https://github.com/philipaconrad/ompt-50-rs) :: Bindings for the first-party OpenMP Tool interface, allowing Rust applications to hook into OpenMP 5.0 runtime events, and change runtime parameters. Used in one of my USC graduate research projects.
* [slackstatus.sh](https://github.com/philipaconrad/slackstatus.sh) :: A short command-line tool to update your Slack status, using the [Slack Presence and Status API](https://api.slack.com/docs/presence-and-status). Made in an afternoon because a colleague put up a bounty for the tool to exist.
* [matrix-archiver-sqlite](https://github.com/philipaconrad/matrix-archiver-sqlite) :: A script to archive content from several [Matrix](https://matrix.org/) chats I care about. Supports incremental backups.


## Earlier Closed Source work

In no particular order:

* _QRECT_ :: A web-based test-taking platform developed by [RCI group](http://www.sc.edu/about/offices_and_divisions/division_of_information_technology/rci/). During a summer there I worked on the original database backend and got the testing infrastructure started for the project.
* _Camp Hill Sensor Firmware_ :: The "Camp Hill v2" was a prototype sensor platform in use at [SDII Lab](http://sdii.ce.sc.edu/) for various research projects. By the time I arrived at the lab, the sensor platform had been commercially abandoned. Over the course of a summer, I reverse-engineered the dual-board system, re-wrote a chunk of the firmware code (fixing a longstanding connectivity bug), and isolated a cause of data corruption to manufacturing defects.
* _Presence Tech_ :: A multi-year contract project with a local startup ([ASSET, LLC.](http://asset-us.com/)) to turn the vibrations from footsteps into position information (think the Marauder's Map from Harry Potter). The project is now exploring industrial and medical applications of the tech. The original codebase was entirely offline, and constructed from a patchwork quilt of applications and programming languages. I developed the current system, which makes analysis results from analog sensors available to application developers in near-realtime. I have also written the entirety of the developer-facing API documentation for the later iterations of the system.

[eopa-styra-docs]: https://docs.styra.com/enterprise-opa/
[gh-eopa]: https://github.com/StyraInc/enterprise-opa
[gh-opa]: https://github.com/open-policy-agent/opa
[opa-docs]: https://www.openpolicyagent.org/
[opa-docs-rego]: https://www.openpolicyagent.org/docs/latest/policy-language/