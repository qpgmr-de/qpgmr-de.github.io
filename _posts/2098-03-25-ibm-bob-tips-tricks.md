## IBM Bob first tips & tricks

Yesterday - on March, 24th 2026 - IBM released Bob, their new AI driven IDE.

And here, I try to compile some first tips and tricks - without any guarantee, that they are working for you or are still valid in the future.

### What's in this fixpack?

As far as I have seen, there are no functional updates. Here is the list of fixes according to IBM:

- the integrated iACS is updated to version 1.1.9.7
- the integrated IBM Java runtime is updated to version 11.0.25+9
- and Security updates

Here the like to the official fix list:

- https://www.ibm.com/support/pages/node/603339


### Problems on the Mac

#### The problem

After you installed IBM Bob it should work for you - the "main user" of the Mac without problems. But sometimes other users on the sdame machine, or even your own user might have the problem, that Bob is not launching correctly.

After you double clicked the Bob icon, you might see a screen that looks like this:

![IBM Bob not launching](/assets/img/2025-03-25-ibm-bob-not-launching.png)

There is no Bob extension in the side bar. No Bob icon in the menu bar. And the Bob chat dialog is not showing up.

I don't know if this is a problem of only me and a handful of other users, or if this is a common problem. But I have seen this problem on my Mac, and I have seen this problem on other Macs, too.

The problem is with the permissions inside the IBM Bob app package.

#### The temporary(!) solution

We hope, that IBM will address the problem with a fix soon - but as long as there is no fix, you can try this workaround:

1. Open the Finder and go to the folder where you installed IBM Bob. It should be in the folder „Applications“.

2. Right click on the IBM Bob app and select „Show Package Contents“.

3. In the folder that opens, navigate to the folder „Contents“, "Resources", "app", "extensions" and lokate the subfolder "bob-code":

   ![IBM Bob app package contents](/assets/img/2026-03-25-ibm-bob-contents.png)
     (I'm sorry that the screen shot is in German as this is my primary language)

4. Now right clich on the folder "bob-code" and select "Get Info" from the context menu.
   ![IBM Bob app package contents](/assets/img/2025-03-25-ibm-bob-permissions-extentions-bob-code.png)
   You see, that "everyone" has "No permissions" ("Keine Rechte" in German) to that folder. That seem to be the problem.

5. Now right click on the folder "Contents" and select "Get Info" from the context menu.
   ![IBM Bob app package contents](/assets/img/2025-03-25-ibm-bob-permissions-contents.png)

5. Now click on the lock icon in the lower right corner of the window and enter your password to make the permissions editable. Change the permissions for "everyone" to "Read only" ("Nur Lesen" in German). Click on small circle with the 3 dots and select the "Apply to enclosed items" and confirm it. This may take some seconds, as the permissions are not written to all subfolders and files.

6. Now close the "Get Info" windows and your Finder window and try start the IBM Bob app again. It should now work.

### *To be continued ...*

I will update this post, when I have more tips and tricks - so stay tuned.