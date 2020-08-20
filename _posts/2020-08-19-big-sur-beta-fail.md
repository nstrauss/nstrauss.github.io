---
layout: post
title: Big Sur Beta 5 - Still Not Education Ready
date: 2020-08-19 20:06 -0500
excerpt: Two months into the beta cycle, Big Sur is still not education ready. Today marks the release of beta 5 and Apple has not implemented a way for standard users to enable screen recording. As a result they can't share their screen with common video conferencing solutions like Webex, Zoom, Google Meet, and others.
---

Two months into the beta cycle, Big Sur is still not education ready. Today marks the release of beta 5 and Apple has not implemented a way for standard users to enable screen recording. As a result they can't share their screen with common video conferencing solutions like Webex, Zoom, Google Meet, and others. Imagine an online class where the teacher isn’t able to display a presentation or a student can't show their work. During the largest shift to remote learning ever, that's the future Apple delivered in beta 1, entirely missing the mark when it comes to how a large majority of their education customers use Macs. Apple has said they have plans to introduce ways for standard users to enable screen sharing, but as of beta 5 that's not the case.

# History and Feedback
Let's take a moment to go over the timeline here.

- June 22. Beta 1 was released.
- June 24. Early beta testers discover standard users can't enable screen recording. Feedback filed.
- June 26. A thread is posted to the developer forums explaining the problem and asking for a response. [https://developer.apple.com/forums/thread/651579](https://developer.apple.com/forums/thread/651579).
- June 26. An Apple engineer responds to my feedback by saying "working as expected."
![Feedback Assistant](/images/sharing_fa1.png)
- June 29. Just kidding! We didn't realize removing an incredibly common workflow would negatively impact customers. Our bad!
![Feedback Assistant](/images/sharing_fa2.png)
- July 22. Beta 3 is released with the note "Admin user credentials are now required to allow Screen Recording in System Preferences > Security & Privacy > Privacy."

That's the last time Apple has commented on screen recording access in Big Sur. No response through feedback assistant, enterprise support cases, or other community channels. After that disappointing turn here's what I wrote to Apple.

"The general wisdom I always hear is filing feedback early in the beta cycle is better. My concern with screen recording in particular is we’ll get a solution in a late beta, it won’t meet the real needs of the user base, and then we’ll be stuck with it. If the solution ships in beta 6, there's no time to provide feedback to improve the feature before GM. Considering Apple didn't even realize this was a problem before we filed feedback, I can't say I'm confident in them developing a solution internally with no feedback from K12 or other education institutions.

Please bump up this feature on the beta roadmap to make it available in beta 4 if that's not already the plan. Specifically I am looking one of these solutions...

- Allow standard users to accept screen recording access without requiring admin rights or being prompted for credentials.
- Add a new preference key to allow standard users to enter their password to accept screen recording access.
- Add screen recording to the list of PPPC payloads (com.apple.TCC.configuration-profile-policy) available for MDM to approve/deny through a profile on supervised Macs."

Again, no response. At this point we don't know if their "alternate solution" will materialize before GM. It could even be in a .1 or .2 release. I hope Apple proves me wrong, but it's late enough in the beta cycle I believe their engineering efforts are focused on fixing bugs. New features have been punted and the first public release of Big Sur will not be usable for most education organizations where their users aren't admins. 

# Apple's Only Exception is Apple
Apple controls specialized access to privileged features through entitlements. Some entitlements developers get for free, they simply need to include them when they build their app. Other more specialized entitlements, security features for example, they need to apply for. And then there's a third category, private entitlements only Apple themselves are able to use. Turns out Apple uses these private entitlements liberally.

Safari includes 80 such entitlements, 43 of which are private. Sounds fair. Among them is `com.apple.private.tcc.allow` with all the goodies included. (Screenshots [Apparency, from the excellent Mothers Ruin Software](https://www.mothersruin.com/software/Apparency/)) Safari gets access to your address book, camera, microphone, calendar, Downloads folder, system events, and the big one, screen recording, all _without your permission_. Yet so far IT admins, even where organizational ownership has already been proven via MDM enrollment, have no option to control these settings.

![Safari](/images/safari_entitlements.png)

Apple's entire privacy model (TCC - transparecy, content, and control) is based around user consent except when it comes to their own apps. And more specific to this post, Apple gets screen recording for free without requiring permission from any user - admin or standard. Try starting a browser based Google Meet or Zoom session in Safari and share your screen. You'd expect to get a TCC prompt like this.

![Chrome TCC](/images/tcc_chrome.png)

But that never happens. Safari doesn't prompt for permission because the private entitlement enables it to bypass TCC altogether. Anyone, even a guest user, could share their screen through Safari without the typical hoop jumping routine required of everyone else. QuickTime and other Apple developed apps get the special private entitlement treatment too.

![QuickTime](/images/quicktime_entitlements.png)

By design Apple won't give TCC approval like screen recording control to MDM administrators, won't allow 3rd party developers to apply for private entitlements, and now won't even let standard users enable screen recording for apps they have control over. With requirements that steep, sometimes it feels like they don't want anyone to share their screen at all. 

# Editorial
How is it possible Apple was this myopic in its understanding of common K12 education workflows? I don't want to make too many assumptions about internal processes I have no insight into. I can only comment as an external observer responsible for managing thousands of Apple devices as my day job. From my vantage point, Apple's enterprise engineering team is extremely insular and literal. When schools with iPads asked to be able to manage Bluetooth settings, an Apple engineering team responded by allowing admins to either restrict or not restrict Bluetooth settings. When the restriction is enabled Bluetooth is stuck in whatever state it was in - either off or on. This workflow has limited usefulness, especially not in a school environment. If forced off, Bluetooth can't be used. Not a bad outcome. If forced on, Bluetooth still can't really be used. Its state is stuck in place. No new devices can be paired, no existing devices can be unpaired. If a students wants to pair new earbuds or a mic they can't. The management available is literal to the letter. Restricted on means restricted. Yet that's the solution Apple engineering provided and called it good. These are the sorts of management tools Apple hands us. Halfway there, never quite what Apple admins need.

