---
title: Google Drive and Mysterious Quota Usage
date: 2020-11-13T19:42:15.000Z
tags: ["Google Drive","quota","files","storage","limbo","shared folders","Google"]
aliases: ["/google-drive-and-mysterious-quota-usage"]
author: Adam
---

So in preparation for Google's [changes](https://www.bbc.co.uk/news/technology-54919165) to Photos and the impact on my storage quota and I went and [checked](https://drive.google.com/settings/storage) what I was using. Turns out my drive storage was using up 9.64Gb, which seemed like a lot, so I compared it to what my locally synced copy was using: 2.3Gb.

Did some Googling and found out that items in your trash count towards the quota, so I went and emptied it. No change. This seems like a problem.

So, I fired up [rclone](https://rclone.org/) in the hope it might be able to shed more light on what was happening than the drive webui:
```
Total:   17G
Used:    4.182G
Free:    10.636G
Trashed: 0
Other:   2.181G
```
Not really.

Some more Googling later I found out that Google Drive does something very stupid. If someone shares a folder with you, and you create things in that folder, then if they delete the shared folder, or any folders within it, anything you created goes into limbo. It still exists and counts against the quota for your drive but isn't visible anywhere because it's not associated with a folder.

If you click on the Storage info in the drive UI they'll show up, but you can't easily separate them from any other files in your drive. If you know the name of specific files you can search for them individually but that's unlikely.

Turns out Google *sort* of address this in a [support article](https://support.google.com/drive/answer/1716222) - they don't really explain how or why the problem occurs but they do tell you how to track down the files: Search for `is:unorganized owner:me`.

Suddenly, there they were. 2.5Gb of files that I never realised were even on my drive, weren't visible in the webui, weren't syncing to my devices, but were definitely being counted against my quota. I moved them all to a temporary folder and then deleted them - remembering to empty the trash - and...
```
Total:   17G
Used:    1.486G
Free:    13.333G
Trashed: 0
Other:   2.181G
```
Much better.

This is clearly atrocious software design by Google; not so much having the files persist in limbo, you probably want to secure against someone deleting a shared folder and all your hard work being nuked, but not surfacing it anywhere. At least have a "Lost and Found" folder that holds all this stuff so it's not just an invisible mass of data using up your storage quota.

Hopefully this has helped and/or very much annoyed someone.
