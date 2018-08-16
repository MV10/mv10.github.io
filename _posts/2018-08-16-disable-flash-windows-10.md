---
title: Disable Adobe Flash in Windows 10
tags: windows flash security edge
header:
  image: "/assets/2018/mm-dd/header1280.jpg"
  teaser: "/assets/2018/mm-dd/header1280.jpg"
---

How to permanently disable Adobe Flash in Edge on Windows 10

<!--more-->

Today we'll take a look at one way to completely prevent Flash from running in Edge on Windows 10. (Why specifically "in Edge"? Chrome has its own separate version using a more secure API, and in spite of this, it is always disabled by default.) We won't _remove_ it because Windows will simply reinstall it during the next update. Instead, we'll use the Windows permissions system to make it off-limits to both updates and installation.

Once upon a time, web browsers were sluggish, and the experience delivered by that slow-motion trainwreck called HTML was even worse than it is today. In 1996, a company named Macromedia acquired rights to a product that was able to deliver higher-quality interactive multimedia content to web users. That product became Flash, and it was a huge success. It was so successful, in fact, that in 2005 Macromedia was absorbed by Adobe. Flash influenced the entire industry. It delivered on the rich-web-content promise where Sun's Java and Microsoft's ActiveX had failed. Microsoft hadn't quite given up yet, however. In 2007 they released Silverlight, an arguably superior product that was stigmatized by poor marketing as a me-too response to Flash.

The web has had a love-hate relationship with Flash, and rightfully so. In the early 2000s at its peak of popularity, many sites were simply non-functional without Flash. Games and high-quality streaming multimedia were suddenly available on-demand in browsers around the world. This was the positive face of a system which also quickly proved capable of delivering remarkably intrusive advertising and damaging malware. Despite having a user-base of over 1 billion installations, thanks to a never-ending stream of vulnerabilities and exploits, the second decade of Flash saw a long, slow, painful decline in popularity.

Fast forward to the present and we find that Flash is (thankfully) in the last stages of life support. In July of 2017, Adobe announced Flash support would end in 2020. Ironically, six months earlier, Microsoft began _forcing_ the installation of Flash in the December 2016 Windows 10 update 1607. (They also did this in Windows 8, but Windows 10 was initially not subject to this imposition.) There is no way to permanently remove it, although Edge does have a disable option buried in the advanced settings page. Unfortunately, Microsoft leaves this virus-vector enabled by default and the average user is none the wiser. To add insult to injury, a few months after the Adobe announcement Microsoft began blogging about their plans to eventually disable and remove this feature they just forced down everyone's throat. I don't know how decisions like these are made at Microsoft, but even if Adobe cut them a big check to spam this garbage around the world (and I sure _hope_ that was the rationale, nothing else makes much sense), it sure looks like someone is asleep at the wheel in Redmond.

**Important:** The changes described in this article require Administrator permissions on your user account (add your user account to the `Administrators` group, or enable the Administrator account and log in with that). Note that I _do not_ guarantee this won't break some future Windows Update. Flash is notoriously buggy and insecure so I assume they will try to update it before we're finally rid of it permanently. If I see issues with a future update cycle, I will add a giant warning to the beginning of this article (and with any luck, a work-around -- I can't fix it until I actually see what happens).
{: .notice--warning}

## The Short Version

In the following sections I'll provide step-by-step instructions and screenshots, but for the technical types, this quick overview may be all you need.

The trick is to simply take ownership of the two install folders and all their contents, then remove _all_ permissions from those folders and files. Additionally, you may wish to do the same to the two files responsible for the Control Panel entry for Flash. Most other instructions I've seen online tell you to remove those files, but here again, Windows will simply reinstall the files making deletion rather pointless.

The folders in question are `c:\Windows\System32\Macromedia` and `c:\Windows\SysWOW64\Macromedia`. The techie-level instructions are:

- Right-click, Properties, Security, Advanced
- Change ownership to the _Administrators_ group
- Replace ownership on subcontainers and objects, apply, close/re-open
- Disable inheritance, replace all child permissions, remove all, apply
- Accept any warnings you might see (never click `Cancel`)

To disable the Control Panel component, apply the same steps to the following two files (obviously there will be no "child object" options since these are not folders):

- `c:\windows\SysWOW64\FlashPlayerApp.exe`
- `c:\windows\SysWOW64\FlashPlayerCPLApp.cpl`

## Detailed Instructions

Open a File Explorer window, make your way to `c:\Windows\System32` and scroll down to the `Macromedia` folder (you can probably just hit the `M` key to jump straight to it). Right-click, choose `Properties`, click the `Security` tab, then click the `Advanced` button.

![01](/assets/2018/08-16/01-right-click-folder.jpg)

![02](/assets/2018/08-16/02-props-security-advanced.jpg)

Notice all those entries with "Full control" or "Execute" access shown in the "Permissions" list. We're basically going to nuke that list and make Flash completely inaccessible to _everyone_ (including you, although that's easy to change, should it become necessary in the future). The first step is to take ownership of the folder (again, we assume your user account is in the `Administrators` group). Click the `Change` link, type `Administrators` in the text box, then click `Ok`.

![03](/assets/2018/08-16/03-owner-change.jpg)

![04](/assets/2018/08-16/04-owner-admin.jpg)

Back in the advanced security dialog, put a check on the newly visible option, `Replace owner on subcontainers and objects`, then click the `Apply` button. Click `Ok` on the pop-up telling you to close and reopen the property page.

![05](/assets/2018/08-16/05-owner-replace.jpg)

![06](/assets/2018/08-16/06-owner-close-reopen.jpg)

Just as instructed, close the advanced security dialog, close the property dialog, right-click the `Macromedia` folder again, click `Properties`, go to the `Security` tab, and click the `Advanced` button again.

![01](/assets/2018/08-16/01-right-click-folder.jpg)

![02](/assets/2018/08-16/02-props-security-advanced.jpg)

This time the advanced security dialog should show `Administrators` as the owner. Click the `Change permissions` button at the bottom. This enables the `Disable inheritance` button and adds a new checkbox with a label that begins with `Replace all child object permissions...` Check the box, then click the `Disable inheritance` button. As soon as you click the button, a pop-up asks what to do with the current inherited permissions. Click the `Remove` option.

![07](/assets/2018/08-16/07-change-permissions.jpg)

![08](/assets/2018/08-16/08-replace-disable.jpg)

![09](/assets/2018/08-16/09-block-inheritance.jpg)

When you return to the advanced security dialog, the permissions list will be empty. You may have noticed "TrustedInstaller" was one of the accounts with "Full control" permissions. That's the account used by Windows Update. If Update can't access the folder at all, it can't change these permissions or re-activate Flash. We aren't done yet, however. If you click the `Effective Access` tab, you'll see a reminder that you still must apply all of these changes, so click `Apply`.

![10](/assets/2018/08-16/10-effective-apply.jpg)

Windows will show a pop-up explaining that you've just removed all permissions. Click `Ok` and Windows will show you _another_ pop-up warning you that you're making system folder changes. They warn you this "can reduce the security of your computer," which is pretty ironic considering we're disabling one of the most bug-laden and insecure pieces of software in computing history. Click `Ok` and rejoice, we're almost finished.

![11](/assets/2018/08-16/11-warning-continue.jpg)

![12](/assets/2018/08-16/12-warning-system.jpg)

## Do Not Enter

To see the effect of these changes, close out of these dialogs, then right-click the `Macromedia` folder, choose `Properties`, and click the `Security` tab. Now you see a message indicating you do not have permissions to view any folder properties.

![13](/assets/2018/08-16/13-no-permissions.jpg)

_Do not_ click on the `Advanced` button. Below, I'll show you what happens if you did, but if you go any further you'll start re-applying permissions to various parts of this folder hierarchy, potentially reducing the effectiveness of the lockdown we just applied. If you were to click `Advanced` you'd see `Unable to display current owner` because you still don't have any permissions here. If you then clicked `Continue` to access the settings with admin rights (again, _do not_ actually do this), Windows would ask you to confirm this change in permissions. 

![14](/assets/2018/08-16/14-view-as-admin.jpg)

Notice the words "permanently" and "folder" in the prompt. This isn't a momentary change in access to work with the advanced security dialog. Continuing will add new permissions to the folder itself. Instead, we'll `Cancel` our way back to our Flash-free existence. 

![15](/assets/2018/08-16/15-open-folder.jpg)

## The Other Windows

64-bit Windows installations also have a folder named `c:\Windows\SysWOW64` which stands for "Windows on Windows64". Rather confusingly, 64-bit DLLs live in the `system32` folder and 32-bit DLLs live in the `sysWOW64` folder. This is to prevent breaking older applications that are sometimes hardcoded to treat `system32` as the only location for system files.

If you navigate to `sysWOW64` in File Explorer, you'll find another copy of the `Macromedia` folder.

![17](/assets/2018/08-16/17-again-for-syswow64.jpg)

As you've probably already guessed, you'll want to reapply the same changes here. During the final "apply" operation, you will probably see the error message shown below. I'm speculating, but I assume the files in `system32` are actually symlinks to the "real" 32-bit files stored here in `sysWOW64`, which is why they're already inaccessible. A symlink (symbolic link) is like a file shortcut you might create on your desktop, except symlinks are a complete stand-in for the "real" file for most filesystem operations (whereas a shortcut can only launch the target file). Simply click `Continue` since this is exactly what we want.

![18](/assets/2018/08-16/18-warning-continue.jpg)

## Take a Test Drive

You can confirm that Flash is no longer available to Edge by navigating the browser to Adobe's Flash test-page at [https://helpx.adobe.com/flash-player.html](https://helpx.adobe.com/flash-player.html). Ironically, if you'd previously disabled Flash in the Edge advanced settings page, Edge will show a one-time pop-up claiming _they_ blocked Flash for your safety. It's sort of sad that Microsoft literally states Flash may be unsafe, and apparently intended to default to a disabled state, even as they forced it on everyone and made it next to impossible to remove.

![20](/assets/2018/08-16/20-ironic.jpg)

As you can see below, after clicking `Check Now` the page will report `Flash Player disabled`, and I've opened the advanced settings page to demonstrate that this is still the case even if Edge is configured to allow Flash.

![19](/assets/2018/08-16/19-flash-disabled.jpg)

## Control Panel

Flash also makes an appearance in your system's Control Panel. We aren't going to remove it since that would require deleting the files, and Windows would just re-install it on the next update cycle. Instead we're just going to disable it. This isn't strictly necessary, but it's easy to do while we're here.

![21](/assets/2018/08-16/21-control-panel.jpg)

The files live in the `c:\windows\sysWOW64` folder where you disabled the 32-bit `Macromedia` folder. You're looking for `FlashPlayerApp.exe` and `FlashPlayerCPLApp.cpl`.

![22](/assets/2018/08-16/22-syswow64-cpl.jpg)

Right-click on `FlashPlayerApp.exe`, choose `Properties`, click the `Security` tab, and click the `Advanced` button. Click the `Change` link, put `Administrators` in the account selection dialog, and when you return to the advanced security dialog, you can click `Disable inerhitance`. You'll be prompted about how to handle the existing inherited permissions -- as with the folders earlier, choose the `Remove` option. Finally, click the `Apply` button and exit the dialogs.

Now go back and do the same thing to the `FlashPlayerCPLApp.cpl` file.

Congratulations. Flash is as completely dead as you can make it right now.

![23](/assets/2018/08-16/23-change-disable-apply.jpg)

## Conclusion

Despite my concern (or worse) with certain decisions Microsoft makes, my friends and colleagues know I'm still something of a Microsoft fan. I've made a good living on their tech stack, they're contribution to computing has been more positive than not (despite the popular rehtoric), and speaking as a developer there is some frankly amazing work happening in the worlds of .NET Core and Azure. But (you knew there'd be a "but") I've always said their marketing people are their weakest link, and everything about this Flash fiasco has "asinine marketing decision" written all over it.

A final reminder -- this scorched-earth approach to disabling Flash may ("probably will") break some future Windows Update cycle. You can't go nuclear without breaking a few eggs. (How's that for mixed metaphors?) I'll update the article if that happens, but if you need an emergency reversal, just go to the two folders and files, grant the `Administrators` group "Full control" access, and delete them. We already know Windows will reinstall them if given the chance -- and if you're here, it isn't as if _you're_ using them.

