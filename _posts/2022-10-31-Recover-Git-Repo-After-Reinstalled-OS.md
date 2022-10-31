---
title: Recover Git Repo after Reinstalled OS
date: 2022-10-31 22:46:22 +0800
categories: [Git]
tags: [how to, git]
img_path: /img/2022-10-31-Recover-Git-Repo-After-Reinstalled-OS/
---

## What's the Problem Like

Let's take a look at the phenomenon of this problem. If you reinstall the operation system with some Git repositories on disk, they will be unusable with the new OS as default. Open the repository with software Git Extension will show like following:

![Corrupted-Repo](Corrupted-Repo.png)
_Corrupted Repository_

It clearly shows that there are some unresolved conflicts existing, and no commit history shows in the center area. Click the 'Resolve' button can open resolve dialog, but there is no conflict listed out.

![Resolve-Conflict-Dialog](Resolve-Conflict-Dialog.png)
_Resolve Dialog_

## How to Solve It

Since there is no conflict to resolve, it's time to try something different for hints. Click the 'Commit' button at the center will pop the commit dialog up. It shows some error log like the diagram below.

![Commit-Dialog](Commit-Dialog.png)
_Commit Dialog_

The first line of the log say that ownership of this repository is dubious, means current active user does not own this repo folder. Right click the repo folder, open its properties page and navigate to security tab like below.

![Properties-Security-Tab](Properties-Security-Tab.png)
_Security Tab_

Enter the 'Advanced' option page, at the top of the new tab shows the owner of this folder.

![Security-Advance-Tab](Security-Advance-Tab.png)
_Advance Tab_

Obviously the owner is a legacy user registed in last operation system, since reinstall the OS does not affect folders and files on other disk partitions. All the folder settings remains the version from old system.

The solution of this issue is clear enough now - get the ownership of this repo folder. Click the 'Change' button highlighted above will open the user pick dialog.

![Pick-User-Dialog](Pick-User-Dialog.png)
_Pick User Dialog_

Type the active user name in the box and click the check button, the active user name will be corrected automatically. Then click the confirm button to select new user. After that, the advance tab will shows the new owner at the top.

![Security-Advance-Tab-After](Security-Advance-Tab-After.png)
_Changes Applied_

> Check the highlighted option to ensure all subfolders and files also apply this ownership changing.
{: .prompt-tip}

Apply the changes and reopen this repo with Git Extension, everything goes okay.

![Recovered-Repo](Recovered-Repo.png)
_Recovered Repository_
