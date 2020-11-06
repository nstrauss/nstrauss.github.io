---
layout: post
title: Deploying Lightspeed Relay Agent with Jamf Pro
date: 2020-11-05 20:44 -0600
excerpt: 'One of the most common questions on the MacAdmins Slack Lightspeed channel is, "How do I install the Relay smart agent on Macs?"'
---

One of the most common questions on the [MacAdmins Slack](https://www.macadmins.org/) #lightspeed channel is, "How do I install the Relay smart agent on Macs?" Lightspeed provides [a little guidance](https://help.relay.school/en/articles/2417658-deploy-the-macos-agent-with-third-party-mdm) and [a decent overview](https://help.relay.school/en/articles/2417746-smart-agent-guide), and that works most of the time, except when it doesn't. Jamf Pro and similar 3rd party management solutions don't handle package installs within DMGs elegantly. This post aims to provide an alternative deployment method that I believe is more reliable, and that I hope Lightspeed eventually considers taking cues from.

# The Installer
Downloading the smart agent from your Relay cloud instance shows a DMG which contains a standard package installer. Take a look at hidden files though to reveal `config.json` which contains a unique hash specific to each customer. It mostly likely translates into a value useful to Lightspeed or correlates to a value in their database. The content can be mostly be ignored, but the file itself is required for install.

![Standard Relay DMG](/images/relay_dmg.png)

There are two problems with the provided install method already. 

1. There's a .json file without any JSON.
2. The JSON file is included dynamically within the DMG for each customer.

I can forgive the first issue since it's more of a preference than a technical error. The second though, bundling customer specific values in the DMG when a standard package is just sitting there, ready to be deployed with Jamf Pro or another 3rd party management solution, is the biggest oversight from Lightspeed. Let's take a look.

![Smart Agent pkg](/images/smartagent_pkg.png)

The regular cast of characters for kext or agent based software - kernel extension, a few launch daemons, and binaries in `/usr/local/bin`. `SmartAgentJS.zip` comes out of nowhere though. Why would a package, whose entire job is to place files on the system, contain a .zip? To the postinstall script.

```ruby
#!/usr/bin/env ruby

# Copyright Lightspeed Systems, 2013 All rights reserved.

#####################################################################################################################
def main
	#copy the smart agent
	system("rm -R /usr/local/bin/SmartAgentJS")
	system("unzip /usr/local/bin/SmartAgentJS.zip -d /usr/local/bin")
	system("cp /Volumes/SmartAgent/.sajs/config.json /usr/local/bin/SmartAgentJS/config.json")
	system("chmod go= /usr/local/bin/SmartAgentJS")
	system("rm /usr/local/bin/SmartAgentJS.zip")
  #create and start our mobilefilter daemon
  system("/usr/local/bin/mobilefilter -d")
end
#####################################################################################################################
main
```

To review, this script...

1. Removes a file that doesn't exist in the package. I assume this is because it could exist from a previous install, and the `rm` command is an insurance policy.
2. Unzip (!) a file containing `smartagentjs`, the main filter agent, and a bunch of other supporting Node.js files.
3. Copy `config.json` from the mounted DMG to that newly unarchived SmartAgentJS folder.
4. Change permissions.
5. Delete the zip file.
6. Start `mobilefilter` binary which also loads the kernel extension.

So many missteps, so little time. Lightspeed needs to seriously overhaul their build process for Relay. Get rid of DMGs, forget the .zip, and move everything to a standard package. If you're a Lightspeed dev reading this, much love for all the work you've already done. You aren't alone as a developer still learning how to deploy enterprise Mac software. Ask a Mac admin sometime how often they deal with similar problems. Grab your favorite drink and take a seat, you'll be there a while. Specifically, the problem here is supporting files - `config.json` and files within `SmartAgentJS.zip` - which need no manipulation at all to be safely included in the package. A better system would be to build a package with those files already in place for each customer, code sign the package, and make it available in the Relay portal. No DMG required. Shift those postinstall script steps to the package build instead of within the package itself to remove complexity, and possible points of failure, on client machines.

The final problem being how the script itself is written in Ruby. Ruby still ships with Big Sur, and as a result this script will continue to work, but Apple [keeps threating](https://developer.apple.com/documentation/macos-release-notes/macos-catalina-10_15-release-notes) to remove interpreters for Ruby and Python in a future version. Using a support, modern shell like `zsh` would future proof the script.

# Repackaging the Installer
To get around these issues I decided to repackage the provided DMG with [autopkg](https://github.com/autopkg/autopkg). [Autopkg](https://github.com/autopkg/autopkg) is an "automation framework for macOS software packaging and distribution, oriented towards the tasks one would normally perform manually to prepare third-party software for mass deployment to managed clients." and the way I deploy most software these days. My [Relay autopkg recipe](https://github.com/autopkg/nstrauss-recipes/tree/master/Lightspeed%20Relay) is publicly available in the [nstrauss-recipes](https://github.com/autopkg/nstrauss-recipes) recipe repo. It automates takeing Lightspeed's provided DMG, ripping it apart, and putting all the required pieces back together with an improved postinstall script. If you're new to autopkg or intimidated by command line tools, don't worry. Using autopkg is surprisingly easy. 

1. Download the latest release of autopkg from [https://github.com/autopkg/autopkg/releases](https://github.com/autopkg/autopkg/releases) and install it.
2. Download the preferred version of the smart agent from [your Relay portal - https://relay.school](https://relay.school)
2. Open Terminal.
3. Run `autopkg repo-add nstrauss-recipes`
3. Run `autopkg run RelaySmartAgent.pkg.recipe --key AGENT_DMG="/Users/bob/Downloads/SmartAgent.dmg"` where the AGENT_DMG path points to the downloaded DMG.

That's it. Check `~/Library/AutoPkg/Cache` for the newly created package.

![Smart Agent pkg](/images/autopkg_relayagent.png)
![Smart Agent pkg](/images/autopkg_relayagent2.png)


Hey look! Our new package has an expanded `SmartAgentJS` folder with your customer specific `config.json` baked right in. And the postinstall script isn't lines of unnecessary Ruby. Hey Lightspeed! Please do this for us. That package can then be uploaded to a distribution point and deployed to Macs.

```
#!/bin/sh

# Set perms on smart agent directory
chmod go= /usr/local/bin/SmartAgentJS

# Start mobilefilter daemon and load the kernel extension
/usr/local/bin/mobilefilter -d
```

Note these directions were only tested on Catalina and above. I've been told by a few people they've had errors on Mojave and earlier.


# Deployment and Reporting
How to deploy the package is up to you. It's included as part of our standard Mac deployment workflow with Jamf Pro. Users are also able to install Relay through Self Service as a troubleshooting step if the agent ever goes sideways. 

![Smart Agent Self Service](/images/relay_selfservice.png)

And though Relay is fairly stable, it does sometimes break in new and exciting ways. When that happens it's helpful to report on the failure before a user notices something is wrong or network traffic is no longer filtered. I wrote a Jamf Pro extension attribute which checks to make sure smart agent processes are running. If they are, it returns the installed smart agent version. If processes are not running it checks to see if the binaries are installed. If any are missing the EA returns "Installed with Errors", and if none of that is true returns "Not Installed". Code is [available here - https://github.com/nstrauss/jamf-extension-attributes/blob/master/relay_status.sh](https://github.com/nstrauss/jamf-extension-attributes/blob/master/relay_status.sh). I created a smart group with "Installed with Errors" or "Not Installed" as criteria and target an install policy to that smart group.

![Relay status Jamf Pro EA](/images/relay_ea.png)

Also be sure there's an approved/allowed kernel extensions profile where Team ID is `ZAGTUU2342` and bundle ID is `com.lightspeedsystems.kext.MobileFilterKext` or else users will be prompted to allow the kernel extension on install.

![Relay approved kernel extension](/images/relay_approvedkernel.png)

# The Uninstaller
You may have noticed Lightspeed includes an uninstall script in their original DMG, but it misses a lot of important files and leaves cruft behind. This uninstall script - [https://github.com/nstrauss/macos-management-scripts/blob/master/uninstall_relay_agent.sh](https://github.com/nstrauss/macos-management-scripts/blob/master/uninstall_relay_agent.sh) - is a more complete version. Use it to remove Relay entirely or as part of a reinstall while troubleshooting. It can be run locally or through a management solution like Jamf Pro without modification.


# The Future
With Big Sur most likely being released sometime next week, Lightspeed still has not released a version using Apple's new [NetworkExtension](https://developer.apple.com/documentation/networkextension) framework. A kernel extension (kext) is still used. Part of the move away from the kernel, NetworkExtensions are a type of [system extension](https://developer.apple.com/system-extensions/) designed specifically to do things like implement an on-device content filter or VPN client. The advantage being system extensions run in user space, not the kernel, and hence are more secure. Probably more important to most K12 admins deploying Relay, starting in Big Sur all software with a kext requires a restart to install.

Let that sink in for a second. In Big Sur, none of your mission critical kext based apps - Relay, Google Drive File Stream, SMART Notebook, etc. - can load their kexts without first restarting. A serious interruption to most deployment workflows, not to mention installing to an already deployed Mac. [This article](https://developer.apple.com/documentation/apple_silicon/installing_a_custom_kernel_extension) describes the process for Apple Silicon, but the same applies to Intel Macs as well. A user has to open Security & Privacy, accept the newly installed kext (referred to as system extension), and then restart. Or as Apple documents:

> Rebooting is a disruptive process for the user, and lowering the security protections may cause concern. Providing a clear explanation of what your kext does, and why its installation is necessary may alleviate some of those concerns. Providing explicit instructions for the user to follow also helps them navigate the reboot process. If you don’t provide these instructions, the user may accidentally or purposefully abort the installation process for your kext.

Disruptive indeed. Apple has budged a bit by allowing standard users to approve new kexts, but the restart requirement remains. System extensions don't have the same consideration. They happily run in user space as long as an approval profile is installed alongside through MDM. Consider whether your org can move to Big Sur if these changes impact you, and ask your Lightspeed contact when you can start testing a NetworkExtension version of Relay.