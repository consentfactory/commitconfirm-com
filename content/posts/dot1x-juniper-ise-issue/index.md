---
title: "Juniper Mist Dot1x User Authetnication Issue"
description: "Exploring the cause and solution of a strange dot1x issue from a Juniper Mist deployment."
date: "2025-04-15"
draft: false
tags: [ juniper, juniper-mist, mist, dot1x, nac, cisco, cisco-ise, ise ]
---
It was a December holiday break, and I was working away, preparing for a coming project, but something stirred in the air; something was wrong. 

{{< img src="attachments/lotrBeaconsLit.gif" alt="lotrBeaconsLit.gif" width="70%" caption="The beacons are lit!">}}

{{< img src="attachments/Chronosphere-20250319_1332.48.gif" alt="We need help with Juniper switching" width="70%" caption="$customer calls for aid!" >}}

{{< img src="attachments/Chronosphere-20250319_1336.47.gif" alt="Jimmy will answer" width="70%" caption="And JImmy will answer." >}}

It appears that $customer is having a problem with *dot1x-hotdesking* (same device, multiple users) on Juniper switching with *new* users. Cisco switching its working fine, but something is up on the Juniper side. If a user's profile has been provisioned already, it works fine; AD-joined Windows devices are working fine as well. 

The cause and solution to this dot1x-hotdesking issue were both fascinating to discover, and the research that I had to perform for this problem really helped me understand what the heck is going on between network switch supplicants, authenticators, and network access control systems. Below is a compilation of my research and what I figured out. 

{{< img src="attachments/benderGetToThePoint.gif" alt="Bender yelling shut up and get to the point" width="70%" >}}

{{< table_of_contents >}}

# tl;dr

## Problem

- Network connectivity fails for new Windows users in hot-desking scenarios (multiple users sharing same system) due to dot1x authentication failure.
- User profile is missing certificate from internal PKI due to network disconnecting after authentication, and therefore fails dot1x authentication.
- Machine and user certificate authentication, on their own, are working normally.

Below is a brief demonstration of normal authentication and the problem this customer had on Juniper switches.
### Normal 802.1x Authentication

{{< img src="attachments/dot1x_general_normal_cx.png" alt="Normal dot1x for user authenticaiton" caption="Right-click and open image in new tab for full size" >}}

### 802.1x - User Cert Missing

{{< img src="attachments/dot1x_general_user-cert-missing.png" alt="User ceritificate missing in dot1x" caption="Right-click and open image in new tab for full size" >}}
## Solution

There are actually a few different solutions to this problem, and some of those are in the links at the bottom. 

This particular solution works by adjusting switch port dot1x timers to fail quickly and move on to MAC authentication. This works by leveraging the machine's previous successful authentication and registration within the NAC (within an endpoint identity group in ISE, for this case), which authorizes the port before the Windows timer resets and tries dot1x authentication again.

Below are the configurations for this. 
### Juniper Mist Switch Configuration

Juniper switch needs the following dot1x configurations.

1. Allow multiple supplicants
2. MAC authentication
3. Adjust RADIUS timers and retries to fail to MAC authentication before 60 seconds. 

In Juniper Mist:

1. Port profile, enable the following:
	- ✅ Allow multiple supplicants
	- ✅ Mac authentication
	- ✅ Port Network (Untagged/Native VLAN) needs to be a network that the user's account can communicate with PKI and obtain a certificate (or you receive a dynamic VLAN from the NAC that accomplishes the same thing)

{{< img src="attachments/juniperMist_portProfile_20250406.png" alt="" >}}

2. RADIUS server profile:
	- ✅ Enhanced Timers

{{< img src="attachments/juniperMist_radius_enhancedTimers.png" alt="Juniper Mist RADIUS server enhanced timers" >}}

The end effect here is that you are causing the dot1x process on the Juniper switch port to fail quickly and move to MAC authentication.

### NAC Configuration

In addition, the NAC solution needs to be configured to allow the MAC address of the endpoint device to authenticate on the port, optionally also sending a tunnel-group-id/VLAN that is within a network that allows the user's account to communicate with the PKI in order to obtain a user certificate. 

That's it. 

{{< img src="attachments/Chronosphere-20250318_1517.55%202.png" alt="GET IN LOSER - WE'RE GETTING NERDY" >}}

# In the Weeds
Alright nerds and AI bots, it's our time. Here's the basic structure of how I put this together.

First off, let me remind you that this is just one solution of many for the Windows dot1x-hotdesking issue, but going through this problem and solution really helps understand the process that happens all the time across vendors.

Here's how this is broken down:

1. Discuss the whole hotdesking problem and briefly show how dot1x works on Windows devices. Or link to someone who already has. 
2. Go through a sequence diagram of the problem itself, explaining where the missing user certificate appears during the process and hwo all the components in the system are part of the process.
3. Go through the switch configurations for both Cisco and Juniper, because it worked on Cisco, so why not Juniper?
4. Another sequence diagram, but this one explaining and showing how *successful* machine-to-user-login dot1x authentication works. 
5. Explain how the network access control (NAC) server (ISE, in this case) acts as part of the whole process, and how group membership works with MAB to assist the Windows machine in getting a server.
6. Give my final thoughts on how to improve the security posture, but at the cost of adding more complexity.

## Breakdown of the Problem and How Windows 802.1x Works

The essential problem of 802.1x authentication in hot desk scenarios on Windows machines is that typically the user profile hasn't been provisioned, let alone the profile provisioned/enrolled with a user certificate. 

To see why, we need to see how the process works. Here's a high-level diagram explaining the process by Jacob Fredriksson. [Fredriksson breaks it down in-detail on their website](https://www.wiresandwi.fi/blog/windows-network-authentication-sequence) .

{{< img src="attachments/windowsNetworkAuthBreakdown.png" alt="Diagram of Windows Network Authentication" caption="Right-click and open image in new tab for full size" >}}

As they explain, the user certificate enrollment process begins *after* a user logs-in for the first time on a machine, which results in an authentication failure because the supplicant has no certificate to send to the NAC, so it sends a null/blank certificate, and of course the NAC with reject the authentication attempt.

## Example: User Certificate Missing, Authentication Rejected

> [!NOTE]
> For all images below, image sizes have been reduced for appearance. Right-click on images and open in new tab for full size. 
>
> Yes, I haven't configured any CSS-javascript magic to do it for you when you click on it. 

Below is an example of this failure scenario that encountered with the following setup:

- Windows 11 Desktop
- Juniper EX 4100 Switch
- Cisco ISE 3.1

0. We assume machine authentication has succeeded (see above) and/or machine network connectivity is functioning normally. 
1. A new user to the machine logs into Windows device and begins user authentication processes. 
2. Wired-AutoConfig restarts the authentication process and ends the EAP session due to the user logging into the device.  
{{< img src="attachments/userCertMissing01.png" alt="" >}}
3. The NDIS authorization state changes from 'Authroized' to 'UnAuthorized'. 
{{< img src="attachments/userCertMissing02.png" alt="" >}}
4. Wired-AutoConfig starts, but because the user does not have a client certificate yet, the Wired-AutoConfig service sends a null/blank certificate list to the NAC. 
{{< img src="attachments/userCertMissing03.png" alt="" >}}
5. Switch forwards  the EAP authentication message to the NAC. 
6. NAC immediately denies the authentication session due to a blank certificate and sends an 'Access-Reject' to the switch because of blank certificate. 
7. Switch receives 'Access-Reject' and puts port in unauthorized state, forwarding response to machine.
{{< img src="attachments/userCertMissing04.png" alt="" >}}
8. Wired-AutoConfig receives EAP failure and immediately suspends any authentication services for 60 seconds. 
{{< img src="attachments/userCertMissing05.png" alt="" >}}
9. Wired-AutoConfig goes to unresponsive state due to timeout. Switch has failure of response from device and waits 30 seconds (by default) before trying again. 
{{< img src="attachments/userCertMissing06.png" alt="" >}}
10. Before switch can retry authentication, the Wired-AutoConfig service timeout expires and Wired-AutoConfig tries authentication again with blank certificate.
11. Steps 5-8 is performed again, and Wired-AutoConfig service re-enters 60 timeout. Steps 9-10 are then performed again, and entire process repeats itself indefinitely until user logs out or receives a certificate.  
{{< img src="attachments/userCertMissing07.png" alt="" >}}

Full version of this here:
{{< img src="attachments/dot1x_detailed_user-cert-missing.png" alt="" >}}

## Switch Configurations 

The customer notified me that while dot1x authentication was working on Cisco switches when users had not yet logged onto the machine. Of course that seems crazy to me because the process is the same on Windows or Juniper, so what's different? 

### Cisco IOS Config

On the Cisco switch side of things, these are the crucial configurations that made authentication work. 

~~~Cisco-IOS
interface GigabitEthernet0/0/1

 ! This line tells the switch to try the next auth method,
 ! MAB in this case, if the first one fails
 authentication event fail action next-method
 
 ! Allows mulitiple supplicants to auth from the same port
 authentication host-mode multi-auth
 
 ! This sets the authentication order, and is crucial in
 ! for this hot desk scenario
 authentication order dot1x mab
 
 ! This allows the user certificate auth to override the MAB
 ! auth once authenitication happens
 authentication priority dot1x mab
 
 ! Gotta enable MAB
 mab
 
 ! Gotta enable dot1x
 dot1x pae authenticator
 
 ! This is secret sauce, because it gives a maximum reauth
 ! attempt at 10 seconds, and IOS will try, by default, 3 times
 ! before moving on to the next auth method
 dot1x timeout tx-period 10
~~~

### Juniper Configuration

There are two places to see the Juniper configuration. I'll show both places: Juniper Mist Wired Assurance, and in Junos CLI.
#### Juniper Mist

Junos breaks down configurations into different protocol components, so unlike Cisco, the configuration is made in a few different locations. In Mist, this configuration happens in both the 'Authentication Servers' configuration, and the 'Port Profile'.

Authentication servers configuration is here:

{{< img src="attachments/authenticationServersEnhancedTimers.png" alt="" >}}

The 'Enhanced Timers' configuration makes the following changes:

{{< img src="attachments/enhancedTimersMist.png" alt="" >}}

> [!NOTE]
> Juniper Mist changed how you can see the configuration changes for items like 'Enhanced Timers', and I think its for the worse. Above is the delta/change output back in November 2024. As of April 2025, it now looks like this:
> 
> {{< img src="attachments/newEnhancedTimersPreviewChanges2025.png" alt="" >}}

The port profile equivalent in Juniper Mist Wired Assurance is below:
{{< img src="attachments/mistWiredAssurancePortProfileDot1x.png" alt="" >}}
#### Junos CLI

Dot1x settings are centrally location in Junos configurations under Protocols > dot1x. The process involves creating a profile with dot1x settings and adding interfaces to that profile. In this case, the logical interface-range interface *dot1x-testing* is added to the profile 'dot1x'.

~~~Junos-Config
authenticator {
    authentication-profile-name dot1x;
    interface {
        dot1x-testing {
            supplicant multiple;
            retries 3;
            quiet-period 2;
            transmit-period 2;
            mac-radius {
                authentication-protocol {
                    eap-md5;
                }
            }
            reauthentication 65000;
            supplicant-timeout 10;
            maximum-requests 3;
            server-fail use-cache;
        }
    }
}
~~~

Junos set commands showing the same thing:
~~~Junos-Set-Commands
set protocols dot1x authenticator authentication-profile-name dot1x
set protocols dot1x authenticator interface dot1x-testing supplicant multiple
set protocols dot1x authenticator interface dot1x-testing retries 3
set protocols dot1x authenticator interface dot1x-testing quiet-period 2
set protocols dot1x authenticator interface dot1x-testing transmit-period 2
set protocols dot1x authenticator interface dot1x-testing mac-radius authentication-protocol eap-md5
set protocols dot1x authenticator interface dot1x-testing reauthentication 65000
set protocols dot1x authenticator interface dot1x-testing supplicant-timeout 10
set protocols dot1x authenticator interface dot1x-testing maximum-requests 3
set protocols dot1x authenticator interface dot1x-testing server-fail use-cache
~~~
## How does it work? Successful Machine-to-User Authentication

Ok, so we've got this hack on the switch side of things to leverage the 60 second timer on the Windows side of things and quickly fail dot1x over to MAB. But how does it actually work?
{{< img src="attachments/howDoesItActuallyWork.gif" alt="" >}}

I'm no **[Veritasium](attachments/https://www.youtube.com/@veritasium)**, but here's the breakdown of how this whole thing works. Prepare your eyes for this eyechart.
 
0. We assume machine authentication has succeeded (see above) and/or machine network connectivity is functioning normally.
1. A new user to the machine logs into Windows device and begins user authentication processes.
2. Wired-AutoConfig restarts the authentication process and ends the EAP session due to the user logging into the device. 
{{< img src="attachments/userAuthSucess01.png" alt="" >}}
3. The NDIS authorization state changes from 'Authroized' to 'UnAuthorized'.
{{< img src="attachments/userAuthSucess02.png" alt="" >}}
4. Wired-AutoConfig starts, but because the user does not have a client certificate yet, the Wired-AutoConfig service sends a null/blank certificate list to the NAC. 
{{< img src="attachments/userAuthSucess03.png" alt="" >}}
5. Switch forwards the EAP authentication message to the NAC . 
6. NAC immediately denies the authentication session due to a blank certificate and sends an 'Access-Reject' to the switch because of blank certificate. 
{{< img src="attachments/userAuthSucess04.png" alt="" >}}
7. Wired-AutoConfig receives EAP failure and immediately suspends any authentication services for 60 seconds.
{{< img src="attachments/userAuthSucess05.png" alt="" >}}
8. Note: NDIS is not dependent on an 802.1x success to be operational and will maintain a DHCP lease if it has one.
{{< img src="attachments/userAuthSucess06.png" alt="" >}}
9. Because Juniper switch is configure with 3 retries and a supplicant timeout of 10 seconds, the switch will fail the dot1x process due to having 3 failures and will move to Mac authentication. 
{{< img src="attachments/userAuthSucess07.png" alt="" >}}
10. The switch forwards the Windows machine MAC address to the NAC for Mac authentication. 
11. The NAC will search its endpoint identity groups. In this instance, it looks up 'internal endpoints' for membership, authenticates and authorizes the endpoint based on policies and sends an 'Access-Accept'.
{{< img src="attachments/userAuthSucess08.png" alt="" >}}
12. Switch marks the port as authorized and maintains the devices active network state.
{{< img src="attachments/userAuthSucess09.png" alt="" >}}
13. Because the network connectivity is functional the machine can communicate with the internal PKI, the user enrolls and obtains a client certificate.
{{< img src="attachments/userAuthSucess10.png" alt="" >}}
14. The Wired-AutoConfig 60 second timer expires and the service tries to authenticate the user with a client certificate from the user. The user has a certificate at this point and so the Wired-AutoConfig service sends the ce rtificate to the NAC.
{{< img src="attachments/userAuthSucess11.png" alt="" >}}
15. Switch receives dot1x authentication for the port and forwards this to the NAC.
16. NAC will check the certificate for authentication, and then will check for authorization of the device. At this point, ISE will either send an 'Access-Accept' or 'Access-Reject'. We're assuming an accept here.
{{< img src="attachments/userAuthSucess12.png" alt="" >}}
17. Switch restarts EAP session with an authorization and forwards the EAP success to the machine. 
{{< img src="attachments/userAuthSucess13.png" alt="" >}}
18. The user's session enters an authenticated and authorized state, and the NDIS Auth State transitions to 'Authorized'.
{{< img src="attachments/userAuthSucess14.png" alt="" >}}
19. Windows machine then proceeds forward with DHCP discovery or normal network connectivity. 
{{< img src="attachments/userAuthSucess15.png" alt="" >}}

The HUGE version of this can be found here:
{{< img src="attachments/dot1x_detailed_success_full.png" alt="" >}}

## NAC MAB Group

The solution used in this post relies on the built-in Cisco ISE local endpoint identity group called 'internal endpoints'. For this solution, when a machine authenticates to the network, the device gets identified and a general ISE profile is made of the device, which by default adds the device to the group 'internal endpoints', which is how MAB/MAC authentication works when RADIUS fails during the 60 second timeout by WiredAuthconfig on the Windows machine. 

Generally speaking, this works, but issue with the 'internal endpoints' group is that it includes *any* device with a MAC address that connects -- all devices that connect are added to the group. 

If your risk tolerance is high, this solution may not work. I have not confirmed this, but I believe a solution for this is to create a specific endpoint identity group for these hotdesk devices. Doing this in an automated fashion on ISE would require the posture licensing, so that may be out of the mix. 

The other approach would be to just manually account for these hotdesk devices and statically assign the devices to this group. However, this adds to the already complex configuration that exists here.

I believe the risk here is small, so I think internal endpoints works fine. 

## Final Thoughts on Process

### Simplicity vs. Complexity

I think the solution identified here is pretty slick for the dot1x-hotdesking issue because it maintains a EAP-TLS configurations and doesn't require extra configuration on the Windows devices, or require reboots on devices, etc. 

However, I think this solution comes at the cost of complexity. Some might consider this a bit of a 'Mouse Trap' solution, but the problem itself is complicated. As a result, the entire solution needs to be properly documented and diagramed to understand the interdependency that exists within the system, which is really no more complicated than any other zero trust security framework.

If this or any other complex system with many interdependcies isn't properly documented, it becomes too much of a snowflake, and the institutional knowledge that someone may have about how it all works becomes a huge risk (aka, the bus factor).

### Improving Network Security of MAB Solution

I think the security of this solution could be improved by assigning VLANs/Tunnel-Group-IDs and sending these in the Access-Accept to the switch so that the MAB device that has some kind of firewalling/access control list in-place to allow the Windows device to perform for just certificate enrollment and any other login services that be needed.

# Bonus Information

## Windows Details

A lot of my time during the initial phase was breaking down the logs used in Windows. Below is some of those notes and research I did. 

### Windows Event Viewer IDs in the Wired-AutoConfig Log

The 'Wired-AutoConfig' log can be found in the following location:

**Event Viewer > Applications and Services Logs > Microsoft > Windows > Wired-AutoConfig**

{{< img src="attachments/Chronosphere-20250318_2053.06.png" alt="Wired-AutoConfig location in Event Viewer" >}}

> [!NOTE]
> Windows has a *wireless* version of this log called 'WLAN-AutoConfig', but the event IDs are completely different.

Below is a list of event IDs from the Wired-AutoConfig log. 
### Event ID Reference Table

| Event ID | Description                                                                                                                                                     |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 14001    | Logged when an existing wired group policy is applied to a computer                                                                                             |
| 14002    | Wired policy is removed from the computer                                                                                                                       |
| 15500    | Logged when a network adapter has been unplugged                                                                                                                |
| 15501    | Logged when a network adapter has been connected                                                                                                                |
| 15502    | Logged when a profile is applied on a network adapter (contains details about profile type and content)                                                         |
| 15504    | Logged when Wired 802.1X Authentication was restarted (includes network adapter details, interface GUID, connection ID, and restart reason)                     |
| 15505    | Logged when Wired 802.1X Authentication succeeded (includes network adapter details, peer/local addresses, connection ID, identity, and authentication results) |
| 15506    | Logged when network authentication attempts have been temporarily suspended on a network adapter (includes reason code and length of block timer in seconds)    |
| 15508    | Logged when there has been an NDIS port state change on a network adapter (shows control state and authorization state)                                         |
| 15511    | Logged when the Wired AutoConfig service entered the running state                                                                                              |
| 15514    | Logged when 802.1X authentication fails (includes failure reason, such as "internal authentication error" or "the authenticator is not present")                |
| 15515    | Logged when the Wired AutoConfig service is starting                                                                                                            |

Generally speaking, the event IDs can have mixed messages in them. For example, 15508 indicates an NDIS Port State change, but that can mean the port is *authorized* or *unauthorized*. For example, see below:

{{< img src="attachments/Chronosphere-20250318_2106.24.png" alt="Event 15508 - NDIS Auth State UnAuthorized" >}}
Event 15508 - NDIS Auth State: UnAuthorized

{{< img src="attachments/Chronosphere-20250318_2109.43.png" alt="Event 15508 - NDIS Auth State - Authorized" >}}
Event 15508 - NDIS Auth State: Authorized

> [!WARNING]
> The event IDs, to the best of my knowledge and searching, are not found in a single location from MIcrosoft. Working with Perplexity to search for these event IDs, they appear to be in various discontiguous locations.
> 
> For example: 
> - **15506** can be found [here](attachments/https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc736026(v=ws.10))
> - **15500** can be found [here](attachments/https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc727808(v=ws.10))
> 
> It's a hot mess:
>
> {{< img src="attachments/Chronosphere-20250318_2117.12.png" alt="Link hierarchy from Microsoft for Windows Server 2008" >}}

## Windows-Juniper EX-ISE Authentication Process

Below is baseline to understand and see how the process works. Yes, some of the images are an eye chart, but I'm trying to compress a lot of data and detail from the whole 802.1x process. 

This is showing a successful machine authentication process, but it looks the same for users (more or less). 

{{< img src="attachments/Chronosphere-20250318_2129.12%201.png" alt="" >}}
1. Windows machine powers on and eventually the 'Wired-AutoConfig' service and authentication process starts.
2. Machine/user applies group policy objects and any other policies relating to 802.1x authentication. As a result of the policies for this machine, 802.1x is enabled, but not enforced on network interface (enforcement corresponds to NDIS control state).
{{< img src="attachments/Chronosphere-20250318_2131.03.png" alt="" >}}
3. Machine/user starts dot1x authentication and begins EAP-TLS authentication, sending certificate to NAC through the switch.
4. The switch (Juniper EX in this case), acts as the network access server and forwards certificate to NAC.
{{< img src="attachments/Chronosphere-20250318_2132.23.png" alt="" >}}
5. NAC (ISE in this case) will check the certificate for authentication, and then will check for authorization of the device. At this point, ISE will either send an 'Access-Accept' or 'Access-Reject'. We're assuming an accept here. 
6. The switch transitions the port to an authorized state and sends an EAP-Success to the Windows machine. 
{{< img src="attachments/Chronosphere-20250318_2133.02.png" alt="" >}}
7. Machine enters an authenticated and authorized state, and the NDIS Auth State transitions to 'Authorized'.
{{< img src="attachments/Chronosphere-20250318_2133.21.png" alt="" >}}
8. Windows machine then proceeds forward with DHCP discovery or normal network connectivity. 

Full version of the diagram:
{{< img src="attachments/Juniper-ISE%20Dot1x%20Machine-User%20Auth%20Issue%20-%20User%20Auth%20Success.png" alt="Full version of the machine/user authentication" >}}

# Links for Further Reading

- **Juniper**
	- [Configure MAC-Based Authentication and MAC Authentication Bypass (MAB) \| Mist \| Juniper Networks](https://www.juniper.net/documentation/us/en/software/mist/mist-access/topics/topic-map/access-assurance-mac-auth-wired-devices.html)
	- [802.1X Authentication \| Junos OS \| Juniper Networks](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/802-1x-authentication-switching-devices.html)
	- [Switch Configuration Options \| Mist \| Juniper Networks](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/reference/switch-config-options.html)
	- Junos CLI
		- [dot1x \| Junos OS \| Juniper Networks](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/dot1x-802-1x.html)
		- [show dot1x \| Junos OS \| Juniper Networks](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-dot1x-802-1x-ex-series.html)
		- [Authentication Session Timeout \| Junos OS \| Juniper Networks](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/authentication-session-timeout.html)
- **Windows**
	- [Configuring Windows Supplicant for 802.1x authentication – integrating IT](https://integratingit.wordpress.com/2019/07/13/configuring-windows-gpo-for-802-1x-authentication/) : How to configure the supplicant in Windows.
	- [Windows Network Authentication Sequence — WIRES AND WI.FI](https://www.wiresandwi.fi/blog/windows-network-authentication-sequence)
	- [Advanced Security Settings for Wired and Wireless Network Policies \| Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/hh994696(v=ws.11)#ieee-8021x---settings)
	- [Managing the New Wired Network (IEEE 802.3) Policies Settings \| Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/hh831813(v=ws.11))
- **Cisco**
	- [Solved: EAP-TLS  - Device vs User certificates - Cisco Community](https://community.cisco.com/t5/network-access-control/eap-tls-device-vs-user-certificates/td-p/4707370)
	- [Configure Cisco ISE and Juniper EX Switches for 802.1X-Based Authentication \| Juniper Networks](https://www.juniper.net/documentation/us/en/software/nce/nce-213_ex_and_cisco_ise/topics/topic-map/nce-213-ex-series-switch-cisco-ise.html)
	- [Configure EAP-TLS Authentication with ISE - Cisco](https://www.cisco.com/c/en/us/support/docs/security/identity-services-engine/214975-configure-eap-tls-authentication-with-is.html)
	- [Troubleshoot Dot1x and Radius in IOS and IOS-XE - Cisco Community](https://community.cisco.com/t5/security-knowledge-base/troubleshoot-dot1x-and-radius-in-ios-and-ios-xe/ta-p/4287439)
	- [Troubleshoot Dot1x on Catalyst 9000 Series Switches - Cisco](https://www.cisco.com/c/en/us/support/docs/lan-switching/8021x/220919-troubleshoot-dot1x-on-catalyst-9000-seri.html)