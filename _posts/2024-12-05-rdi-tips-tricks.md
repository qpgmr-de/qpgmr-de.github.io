## Rational Developer for i 9.8 - Tips and Tricks

With the availability of RDi 9.8.0.3 and the latest compiler PTFs I updated my 
compiled list of RDi 9.8 tips and tricks that piled up since the first beta. 

I decided to also post it to the 
[IBM TechXchange community](https://community.ibm.com/community/user/power/blogs/daniel-gross/2024/12/03/rdi98-tips).

So after using version 9.8 since the BETA releases, I have compiled my favorite
information, tips and tricks for it. This compilation is a very personal and 
loose collection - without claiming any completeness or that it will work 
"on your computer". As always, your mileage might vary.

### Table of contents

- [RDi 9.8 has a completely new installation process](#rdi-98-has-a-completely-new-installation-process)
- [Applying the permanent activation kit license](#applying-the-permanent-activation-kit-license)
- [Using a RDi 9.6 workspace with RDi 9.8](#using-a-rdi-96-workspace-with-rdi-98)
- [Installing RDi 9.8 updates (like 9.8.0.3)](#installing-rdi-98-updates-like-9803)
- [Installing iSphere and other plugins](#installing-isphere-and-other-plugins)
- [Solving remote system password problems](#solving-remote-system-password-problems)
- [Bringing back Snippets](#bringing-back-snippets)
- [Using RDi in an other language](#using-rdi-in-an-other-language)
- [Fixing the „lagging“ in compiler error list view](#fixing-the-lagging-in-compiler-error-list-view)
- [Conclusion](#conclusion)

### RDi 9.8 has a completely new installation process

Yes - RDI 9.8 doesn't use the IBM Installation Manager (IM) anymore. RDi 9.8 is 
using the typical Eclipse installation process called "P2". 

Simply spoken, on Windows you  only have to decompress the ZIP file "IBM Rational
Developer for i.zip" to any location where you have write permissions and start 
RDi by double-clicking "RDi.exe". On macOS you simply copy the "IBM Rational 
Developer for i.app" bundle to a location of your choice and double-click the 
application. That's it.

The nice thing - you can keep your RDI 9.6 installation in parallel and you can 
even have multiple copies of RDi 9.8 if necessary.

### Applying the permanent activation kit license

Not only the installation itself and the updates process are new - applying the 
license also works different than in 9.6. But I can only document the process 
using an activation kit, because I don't have a license server available.

After lauching RDi 9.8 simply select the trial license - don't use the "Manage 
Licenses" dialog. Instead select "Install new software..." from the "Help" menu. 
Click on the "Add..." button - give it a more or less meaningful name and select 
"Archive...".

![RDi activation kit](/assets/img/2024-12-05-rdi-activation-kit.png)

Here you should select the file „RDi_9.8_RPGCOBOLTools_AU_Lic_Act_Kit_P2.zip“ 
which is included in the file „RDi_Act_Kit_9.8_MP_ML.zip“ (which you have to 
unpack).

![RDi activation kit installation](/assets/img/2024-12-05-rdi-activation-kit-install.png)

Select the permanent activation kit and install it. RDi has to restart to 
apply the license.

You only have to apply the 9.8 license once – any new installation of RDi 9.8 
uses the same license on that machine.

### Using a RDi 9.6 workspace with RDi 9.8

I have used a RDi 9.6 workspace – used it in RDi 9.8 – used it again in RDi 
9.6 – back and forth – and I never had any problem with that. Of course you 
cannot use the same workspace in RDi 9.8 and 9.6 at the same time.

So I never had any problems – but your mileage may vary – so there is no 
guarantee that at one point your workspace will get corrupted and will be 
unusable. So better make backups of your workspace regularly.

### Installing RDi 9.8 updates (like 9.8.0.3)

After the installation there should already be a preconfigured update site for 
RDi 9.8 updates – just select „Install new software…“ in the „Help“ menu. Then 
select the preconfigured RDi update site:

![RDi update repository](/assets/img/2024-12-05-rdi-update-repository.png)

If there are any updates available – you can check the updates you want, click 
the „Next >“ button and follow the instructions..

If there is no preconfigured update site for RDi (whether it was deleted or 
anything) you can click on the „Add…“ or „Manage…“ button and create an update 
site. Give it a meaningful name, and enter the following URL in the 
„Location“ field:

- IBM Rational Developer for i update site

  [https://public.dhe.ibm.com/ibmdl/export/pub/software/awdtools/rdi/v98/](https://public.dhe.ibm.com/ibmdl/export/pub/software/awdtools/rdi/v98/)
  
If you want information about the upfdates, just go to Fix Central:

- IBM Fix List for RDi

  [https://www.ibm.com/support/pages/fix-list-rational-developer-i](https://www.ibm.com/support/pages/fix-list-rational-developer-i)
  
When this update site is selected, the RDi 9.8 updates should be presented. 
Check the updates you want, click the „Next >“ button and follow the 
instructions.

Sometimes updates, features or plugins are not code signed – then just click 
„Select All“ and „Trust Selected“:

![Trust unsigned content](/assets/img/2024-12-05-trust-unsigned.png)

But always look through the list of classes and versions to be sure, that you 
don’t install any potentially harmful software.

Alternatively you can also download the fixpack from Fix Central and add the 
downloaded ZIP file to your update sources.

Select „Install new software…“ from the „Help“ menu. Click on the „Add…“ 
button – give it a more or less meaningful name and select „Archive…“. 

Select the downloaded ZIP file and add the repository. After this, the 
process is the same.

### Installing iSphere and other plugins

To install the following popular plugins simply follow the instructions how 
to an update site is configured in the previous paragraph.

- iSphere – the RDi users „Swiss-army-knife“ plugin

  [https://rdi-open-source.github.io/isphere//update-site/eclipse/rdi8.0](https://rdi-open-source.github.io/isphere//update-site/eclipse/rdi8.0)
  
- iRpgUnit – unit testing for RPG

  [https://tools-400.github.io/irpgunit//update-site/eclipse/rdi8.0](https://tools-400.github.io/irpgunit//update-site/eclipse/rdi8.0)
  
- Rapid-Fire – fast large database migration tool

  [https://rdi-open-source.github.io/rapid-fire//update-site/eclipse/rdi8.0](https://rdi-open-source.github.io/rapid-fire//update-site/eclipse/rdi8.0)

### Solving remote system password problems

If you have problems connecting to your systems, this might be related to 
different encryption algorithms from RDi version 9.6 and version 9.8.

The best way solve this problem is to disconnect from all systems, and clear 
the passwords.

![RSE clear passwords](/assets/img/2024-12-05-rse-clear-passwords.png)

After that, don’t re-connect to the systems immediately.

Instead select „Preferences“ in the „Window“ menu, and search for 
„password“ – then select „Secure Storage“.

![Preferences clear passwords](/assets/img/2024-12-05-preferences-clear-passwords.png)

Here click „Clear Passwords“ again – and click „Change Password…“ while 
„Windows Integration (…)“ is selected. You can provide a password hint, but 
you don’t have to.

Apply and close the settings and connect to your systems. You have to enter 
your passwords again, and check „Save password“. Now it should work again.

### Bringing back Snippets

Snippets is a feature, that some users relied on for their development. 
But RDi 9.8 doesn’t install Snippets and also the feature to distribute 
Snippets from a system was removed.

At least, it is able, to bring back Snippets by installing the Eclipse Web 
Developer Tools package. But – RDi is lagging behind the Eclipse IDE 
development, so you can’t simply install the Web Developer Tools from the 
Eclipse Marketplace.

So here what you have to do – once again select „Install new software…“ in 
the „Help“ menu and create a new update site by clicking the „Add…“ button. 
Give the new site a meaningful name and enter the following URL in the 
„Location“ field:

- Eclipse 4.23 software repository

  [https://download.eclipse.org/releases/2022-03](https://download.eclipse.org/releases/2022-03)

Use the search box to search for „Web Developer Tools“ and set a check mark 
in front of „Eclipse Web Developer Tools“. Follow the installation after 
clicking the „Next >“ button.

![Eclipse Web Developer Tools](/assets/img/2024-12-05-eclipse-web-developer-tools.png)

If you had the „Snippets“ view open, it should work after RDi has restarted. 
If you closed the non-working view, you can now open it by selecting 
„Show View“ from the „Window“ menu. Search for „Snippets“ and open the view 
again.

### Using RDi in an other language

You can use RDi in an other language than the language of your installed OS. 
As an example I use Windows in German but I want to use RDi in English.

Simply open the file „RDi.ini“ in the installation directory of RDi, and add 
a line with „-nl“ and another line with the language you want to use – here 
I selected „en“ for English:

![RDi.ini](/assets/img/2024-12-05-rdi-ini.png)

Save the file and start RDi – and the UI will be in the selected language, 
as far as this language is available. If some elements are not translated, 
they will show up in English – which make a funny mix.

### Fixing the „lagging“ in compiler error list view

Since RDi 9.8 some users (including me) have encountered a strange „lag“ 
when double-clicking on an error in the compiler error list view. This 
double-click should bring you to the corresponding source line in LPEX – 
but sometimes it freezes the RDi UI for several seconds before jumping to 
the error.

Right now there seems to be no fix for RDi – but installing the latest 
compiler PTFs and setting an environment variable during compilation, seem 
to fix it from the server side. The ILE RPG compiler has PTFs for target 
releases V7R4M0 and V7R5M0 to support a compile-time environment variable to 
prevent RNF7031 messages from being placed in the event file.

```
ADDENVVAR QIBM_RPG_RNF7031_EVENTS VALUE('N')
```

You could add this ADDENVVAR command in the „Initial command:“ section in 
the Properties for your RDi connection.

If you compile in batch from RDi, you should also add CPYENVVAR(*YES) in the 
„SBMJOB additional parameters“ in the „Commands“ tab of the Properties for 
your connection.

The PTFs are:

- Version 7.5 TGTRLS(*CURRENT) – 5770WDS SJ02922
- Version 7.4 TGTRLS(*CURRENT) – 5770WDS SJ02501
- Version 7.5 TGTRLS(*PRV) – 5770WDS SJ02923

With this fix, RDi should load the compiler error list view quicker and also 
react promptly and without „lagging“ to a double-click in the compiler error 
list view.

### Conclusion

RDi 9.8 is a major update and brings a completely new installation and update 
process – but this new process is much better in line with traditional 
Eclipse processes.

One problem still persists – the Eclipse version on which RDi is based lags 
several versions (or releases) behind the original Eclipse IDE. But at least, 
RDi has catched up a little, so it gets easier to use other Eclipse features.

Many Thanks to the team at Fortra for putting so much effeort into the RDi 
development – and very special thanks to Steve Ferrell for his support 
during the beta period.
