---
layout: post
title: Mitigating Mac Enrollment Failures
date: 2020-08-11 16:47 -0500
excerpt: While working to enroll 1,000+ Macs to prep for the start of school, we found a large number were failing to get an enrollment configuration during Setup Assistant. There were three distinct ways the process failed.
---

<!-- ![_config.yml]({{ site.baseurl }}/images/config.png) -->

While working to enroll 1,000+ Macs to prep for the start of school, we found a large number were failing to get an enrollment configuration during Setup Assistant. There were three distinct ways the process failed.

1. The Remote Management screen didn't appear during Setup Assistant. The Mac never got its MDM defined enrollment configuration and setup continued as a consumer device. The next screen after the country chooser was Data & Privacy.
2. Setup Assistant was stuck at "Retrieving activation record."
3. The Mac attempted to get an enrollment configuration and errored out with "Unable to configure your Mac. An error occurred while obtaining configuration settings. Your Mac will not be configured with default settings from (null). To apply these settings later, click Continue."

After a fresh OS install, around 80% were failing to enroll under one of these scenarios. Rough. A project scheduled to take a week was about to take much longer, and during maybe one of the busiest summers in a typically already busy time of year, I needed a solution.

![UCRT](/images/retrieving_action_record.png)

# The Problem
After consulting the Google machine it turns out others had the same problem.

[https://www.jamf.com/jamf-nation/discussions/33723/catalina-prestage-enrollment-retrieving-activation-record-hanging](https://www.jamf.com/jamf-nation/discussions/33723/catalina-prestage-enrollment-retrieving-activation-record-hanging)

Not only was this impacting others, it had been a known issue since October 2019. Another colleague I spoke to had reported the bug to Apple in Feburary 2020. Great, a longstanding bug affecting enrollment and Apple has effectively not shipped a fix for six point releases. Most comments in the Jamf Nation thread pointed to a post by Graham Pugh. 

[https://grahamrpugh.com/2020/02/21/resetting-dep-without-reinstalling.html](https://grahamrpugh.com/2020/02/21/resetting-dep-without-reinstalling.html)

Graham explains how to get a root Terminal session during Setup Assistant. 

1. Boot to recovery.
2. Run `touch /var/db/.RunLanguageChooserToo` to force the language chooser.
3. Restart.
4. Select control + command + option + T at the language chooser to get a root Terminal session.
5. Run `profiles renew -type enrollment` (or the legacy `profiles -N`) to force an enrollment configuration check.

In most cases you should then be able to click through Setup Assistant and successfully enroll the Mac. However, this is slow and requires manual intervention, and even then `profiles` wasn't always working. Sometimes it would take multiple runs to finally trigger the expected Remote Management screen. With over 1,000 Macs to enroll, I didn't have the time to fiddle about with it. So off to enterprise support I went.

AppleCare enterprise support identified the bug quickly after sending over a few example sysdiagnose(s). They went straight to the logs to explain why enrollment was failing.

```
dasd com.apple.duetactivityscheduler scoring 0:com.apple.mobileactivationd.UCRT.OOB:D65047:[
{name: BatteryLevelPolicy, policyWeight: 1.000, response: {Decision: Must Not Proceed, Score: 0.00, Rationale: [{batteryLevel == 1}]}}
{name: NetworkQualityPolicy, policyWeight: 8.400, response: {Decision: Must Not Proceed, Score: 0.00, Rationale: [{[wiredQuality]: Required:20.00, Observed:0.00},{[wifiQuality]: Required:50.00, Observed:0.00},{[networkPathAvailability]: Required:1.00, Observed:1.00},]}}
{name: ThermalPolicy, policyWeight: 5.000, response: {Decision: Absolutely Must Not Proceed, Score: 0.00, Rationale: [{thermalLevel >= 2}]}}
], FinalDecision: Absolutely Must Not Proceed}
```

[Duety activity scheduler (DAS)](https://eclecticlight.co/2017/05/23/how-macos-runs-background-activities-4-using-xpc-activity/) run by [dasd](https://developer.apple.com/documentation/foundation/nsbackgroundactivityscheduler) is responsible for scheduling a list of scored tasks. Scores are based on battery level, network quality, and thermal state. If the composite of those weighted scores exceeds a specific threshold, the task doesn't run - "Absolutely Must Not Proceed." The above log shows DAS scoring `com.apple.mobileactivationd.UCRT.OOB`, a system launch daemon task to get a UCRT. I still don't fully understand what UCRT is, but I assume it's required for certificate based authentication to an Apple service or MDM. The important bit is a UCRT is required for `profiles renew -type enrollment` to succeed. Requesting an enrollment configuration without UCRT, and subsequently getting through the Remote Management screen at Setup Assistant, fails.

![UCRT](/images/com.apple.mobileactivationd.UCRT.OOB.png)

```
teslad com.apple.CloudConfigurationDaemon Default Error while retrieving client certificates: Error Domain=com.apple.MobileActivation.ErrorDomain Code=-4 "UCRT is unavailable." UserInfo={NSLocalizedDescription=UCRT is unavailable., NSUnderlyingError=0x7fc3c0418950 {Error Domain=com.apple.MobileActivation.ErrorDomain Code=-1 "Error occurred during request: Failed to retrieve UCRT. ({
NSLocalizedDescription = "Failed to retrieve UCRT.";
NSUnderlyingError = "Error Domain=com.apple.MobileActivation.ErrorDomain Code=-4 \"UCRT is unavailable.\" UserInfo={NSLocalizedDescription=UCRT is unavailable.}";
})" UserInfo={NSLocalizedDescription=Error occurred during request: Failed to retrieve UCRT. ({
NSLocalizedDescription = "Failed to retrieve UCRT.";
NSUnderlyingError = "Error Domain=com.apple.MobileActivation.ErrorDomain Code=-4 \"UCRT is unavailable.\" UserInfo={NSLocalizedDescription=UCRT is unavailable.}";
})}}}
```

And it will fail over and over again. `teslad` is the underlying process run in association with `profiles renew -type enrollment` and similar `profiles` commands. And without UCRT, `profiles` is useless for our purposes. More frustrating is `teslad` will attempt to get an enrollment configuration multiple times, but exhausts its retry attempts within a short window. It does not retry on restart or by sitting long enough to the best of my knowledge, at least not in a useful, reliable way.

```
2020-08-06 09:58:35.445081 -0700 teslad com.apple.CloudConfigurationDaemon Default Retrying the request
2020-08-06 09:58:39.467454 -0700 teslad com.apple.CloudConfigurationDaemon Default Retrying the request
2020-08-06 09:58:40.408524 -0700 teslad com.apple.CloudConfigurationDaemon Default Retrying the request
```

If UCRT isn't available during retries, the Mac's chance to get an enrollment configuration without additional intervention is gone. Run `profiles show -type enrollment` to attempt to print the Mac's enrollment configuration without UCRT present and it will return a 34006 error - "The cloud configuration server is unavailable." 

```
Error fetching Device Enrollment configuration: (34006) Error Domain=MCCloudConfigurationErrorDomain Code=34006 "The cloud configuration server is unavailable." UserInfo={NSLocalizedDescription=The cloud configuration server is unavailable., CloudConfigurationErrorType=CloudConfigurationFatalError}
```

Why then were so many of my Macs not getting UCRT? The logs showed they were too hot. DAS consistently chose not to run the UCRT task because it scored thermals too high. My workflow to reliably trigger an "Absolutely Must Not Proceed" score from DAS was very simple.

1. Plug the Mac into power.
2. Boot to recovery. 
3. Use [MDS](https://twocanoes.com/products/mac/mac-deploy-stick/) to wipe the drive and reinstall macOS.

The heat generated from installing macOS while the battery charged in a 70° F room was enough to cause enrollment to fail. Most 2020 MacBook Air got so hot my testing showed the average CPU die temperature around 100° C. To make matters worse, there is no way to run a UCRT request directly. An Apple engineer might know the magic to request UCRT on demand, but that information isn't available to admins. The only way to get UCRT is to wait for DAS to return a score sufficient for `com.apple.mobileactivationd.UCRT.OOB` to run and then trying `profiles renew -type enrollment` again. That means waiting for the 100° C CPU to cool down.

```
2020-08-06 10:03:07.321302 -0700 dasd com.apple.duetactivityscheduler scoring 0:com.apple.mobileactivationd.UCRT.OOB:D24905:[
{name: MemoryPressurePolicy, policyWeight: 5.000, response: {Decision: Can Proceed, Score: 0.50, Rationale: [{[memoryPressure]: Required:2.00, Observed:1.00},]}}
{name: ThermalPolicy, policyWeight: 5.000, response: {Decision: Can Proceed, Score: 0.20, Rationale: }}
] sumScores:32.210000, denominator:38.910000, FinalDecision: Can Proceed FinalScore: 0.827808}

2020-08-06 10:03:07.621823 -0700 mobileactivationd <Unknown> Performing UCRT OOB.
2020-08-06 10:03:11.122994 -0700 mobileactivationd <Unknown> Successfully performed UCRT OOB.
```

Here we can see DAS finally giving the task a passing score - "Can Proceed" - and `mobileactivationd` gets UCRT. `profiles renew -type enrollment` can now be run successfully.

# The Mitigation
[https://github.com/nstrauss/macos-management-scripts/tree/master/enrollment_config_helper](https://github.com/nstrauss/macos-management-scripts/tree/master/enrollment_config_helper)

I wrote the best mitigation script I could within time constraints. The script will...

1. Print basic system info (OS version, build, etc.) to a log.
2. Use `powermetrics` to print CPU and IO thermal level, fan RPM, CPU die temp, and overall thermal pressure to the log.
3. Run `profiles renew -type enrollment` to attempt to get an enrollment configuration.
4. If successful, print the enrollment configuration info and move on. 
5. If it fails, wait 5 seconds and try again. 
6. If it fails after the max retry attempts (default 3), wait one minute to give the Mac time to cool down before trying again. 
7. Rinse and repeat for 10 cycles (configurable) until the Mac finally gets an enrollment configuration. 
8. If after all cycles complete and the Mac still can't get its config, print a message recommending it be shut down for 10 minutes and then try again.

The time savings comes in that the script is run as root with an accompanying launch daemon. There's no need to get a root Terminal session or run commands manually. Here's an example of the script's log output. In my environment getting an enrollment configuration almost always succeded on cycle 2, retry 1 - the 4th attempt about 2-3 minutes after boot.

{% gist 86bca3bd2b361f97f7a48cfe3e56d07a %}

You can see how immediately after boot post-OS install the CPU runs at 100° C. A few minutes later it's at a much cooler 66° C, DAS has allowed the UCRT task to run, and `profiles` does its magic. One more restart (to restart the Setup Assistant process mostly) and the Mac will reliably enroll.

# Implementing with MDS
Getting the script and launch daemon place alongside a new OS install is fairly easy by taking advantage of the `startosinstall` option `--installpackage`. [MDS](https://twocanoes.com/products/mac/mac-deploy-stick/) makes this all configurable through button clicking, but if you already have your own `startosinstall` workflow built out the package can be included.


1. Download and install MDS. Make sure you have an OS installer available.
2. Create a new workflow.
![mds](/images/mds_workflow1.png)
3. Select an OS installer, erase and rename the drive if preferred. 
![mds](/images/mds_workflow2.png)
4. Select resources. Include a package with `enrollment_config_helper.sh`, the launch daemon, and a postinstall script to load the daemon. Example launchdaemon and package are available here - [https://github.com/nstrauss/macos-management-scripts/tree/master/enrollment_config_helper](https://github.com/nstrauss/macos-management-scripts/tree/master/enrollment_config_helper). You can download, unzip, and use the package as is as long as you're fine using my internal naming conventions. None of it is signed or notarized as that isn't a requirement of `startosinstall`. Alternatively you can also put the pieces together into a package yourself.
![mds](/images/mds_workflow3.png)
![mds](/images/mds_workflow4.png)
![mds](/images/mds_workflow5.png)

That's the entirety of the workflow. Save to volume, disk image, or other MDS run method. I prefer to save to a disk image to serve off a web server. The script and launch daemon (or prebuilt package) can also be included in your existing `startosinstall` outside of MDS. I'm using MDS here because it's easy and others might already have familiarity with the tool. If using Mac Provisioner or another wrapper around `startosinstall` there's probably a way to configure it to include the package during OS install.

MDS will wipe the drive, use `startosinstall` to reinstall the OS and `enrollment_config_helper.pkg` alongside it. When the Mac boots for the first time to Setup Assistant follow these steps.

1. Select control + option + command + T to pop a Terminal window. This isn't a root Terminal and doesn't need to be. 
2. Run `tail -f /var/tmp/mds_dep_enroll.log` to view the script's log file.
3. Wait for `profiles renew -type enrollment` to successfully get an enrollment configuration. For an example of what that typically looks like see the log file above.
4. Restart the Mac once the MDM enrollment configuration prints to the log. It will be obvious as it's formatted text with your organization name, contact info, etc.
5. After restart, click through Setup Assistant as usual and the Mac will enroll.
6. Success!

# Editorial
That any of this is necessary at all is a failing on Apple's part to properly triage and prioritize bugs impacting core Mac management features. The bug was first reported to Apple in October 2019 soon after Catalina was released. The best response Apple enterprise support has been able to provide from engineering is that they're planning to land a fix in Big Sur. It shouldn't take an entire major OS release cycle. If Apple wants to be enterprise ready it needs to put more engineering effort towards fixing bugs like this one. Apple, along with its MDM partners, market "zero touch deployment" where an IT department can dropship a shrinkwrapped Mac to be enrolled and corp managed easily by any employee. The reality is much different. Enrollment is fragile and prone to breaking, with no guarantee a Mac will be enrolled at all before landing at the desktop. As my experience shows, a too hot Mac can stop enrollment in its tracks, completely erasing the frictionless experience admins have been promised for years. I'm more of the opinion enrollment should be like the USPS - neither snow nor rain nor heat nor gloom of night nor overheating CPU - shall keep it from happening. Apple has promised a fix in place soon, but it feels like too little too late after fighting through 1,000 enrollments in a week.