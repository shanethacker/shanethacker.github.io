---
layout: post
title: Fixing Skipped Updates in JIRA Agile
category: tools
tags: [jira,server]
---

Just a quick post to get a solution out there for a weird problem that apparently only I've had.

Over the weekend, I worked on the long-overdue production JIRA upgrade at my work. (Seriously, we were three major versions behind. By the time I became the admin, we were all afraid to touch it for fear of losing our precious workflows.) I had tested the upgrade process on another server, taking that one from 5.2.7 to 7.12 fairly smoothly, except for when it became apparent our test VM needed more than 2GB of RAM. So, forewarned about add-on incompatbilities and upgrade paths, I was reasonably confident I could handle any upgrade weirdness.

What I wasn't expecting was for JIRA to refuse to recognize a step in the upgrade process, meaning the carefully planned 5.2.7 to 6.4 to 7.0 to 7.12 path left some pieces behind when the 6.4 upgrade didn't register or run some of the upgrade steps, even though it appeared to be working when tested. Specifically, the Greenhopper/JIRA Agile/JIRA Software transition only about half worked.

The Symptom: We found our Agile boards had disappeared in 7.12.

The Cause: The JIRA Software application wasn't starting up due to failed upgrades.

The Log Entry:

```
2019-08-11 10:55:54,059 JIRA-Bootstrap ERROR      [c.a.s.core.upgrade.PluginUpgrader] Upgrade failed: This version of JIRA Agile requires a minimum build number of #45 in order to proceed. The latest upgrade task to have been successfully completed is #39. You must uninstall this version of JIRA Agile and attempt an upgrade via an intermediary version.
```

The Reason: Atlassian stopped providing cumulative updates from the beginning of time in Greenhopper plugin 7.1. The 7.12 version of Greenhopper could only start the upgrade process at step 45. We were at step 39.

Suggested Solution: Replace the Greenhopper plugin with the last version that had all cumulative updates, 7.0.11. Restart the server and watch the updates fly. Very reasonable. Easy to find on the Atlassian site.

Result: Failure.

The Symptom: No upgrades. No boards.

The Cause: JIRA Agile never starts up to try to upgrade anything.

The Log Entry:

```
'com.pyxis.greenhopper.jira' - 'JIRA Agile'  failed to load.
    		Cannot start plugin: com.pyxis.greenhopper.jira
    			Unresolved constraint in bundle com.pyxis.greenhopper.jira [185]: Unable to resolve 185.0: missing requirement [185.0] osgi.wiring.package; (&(osgi.wiring.package=net.java.ao)(version>=0.9.0)(!(version>=2.0.0)))
```

Suggested Solution: None. That's weird.

Desire: Not to roll back to the beginning and disrupt all of the users that don't need the Agile Boards.

Next Step: Digging through documentation and muttering.

The Find: [JIRA's OSGI Browser](https://developer.atlassian.com/server/framework/atlassian-sdk/using-the-osgi-browser/)

The Clue: The OSGI browser made one thing obvious. JIRA 7.12 uses net.java.ao 2.0.0, making that part of the log entry very meaningful.

The Question: What happens if I grab the last version of net.java.ao from the JIRA upgrade backups and temporarily replace the bundled plugin, while running Greenhopper 7.0.11? (Always say yes to backups.)

Result: Me watching the build number tick up one by one in the database. Yes!

Cleanup: Replace the replaced files with their correct versions, restart, and watch the build numbers hit 51. Reindex, and we're golden.

Lessons Learned: Update JIRA often, and always pay attention to log files.
