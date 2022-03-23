---
title: "Autopilot Group Tag Deep Dive"
date: 2022-03-21T19:07:10Z
draft: true
---

“Group Tag”: One of Autopilot’s hidden gems
In our modern managed projects, especially while leveraging native Azure AD joined devices, we typically conclude that: 

The customer rarely has a traditional hierarchical OU structure, containing DTAP-, device type- and/or location information.
We really like the option to validate settings and/or applications to a subset of the environment (call it pre-prod, canaries or guinea pigs, …) before applying these to the entire estate. 
We want to keep the device names as simple/default as possible (since each exception to the device naming template implies more device enrolment profiles, maintenance and complexity).
As most people reading this blog will undoubtfully know, Windows autopilot is the modern deployment method for corporate Windows 10 devices. It allows an enterprise to pre-register corporate owned Windows 10 devices (using their unique hardware hashes) and link these devices to their Azure AD tenant. Once the device is known to the tenant, it is considered a trusted device that can be staged (enrolled) using autopilot.

One of the most underestimated powers in this Autopilot story today, I believe, is the “Group Tag” attribute in the autopilot service.


This tag can be provided during the pre-registration (import in Autopilot) but can easily be set or modified in a later stage as well.

During the initial import in Autopilot, an Azure AD device object is created automatically, and the “Group Tag” attribute is applied to the corresponding Azure AD device object too, and well before actually staging/enrolling the device! 

Note that in AAD, however, the attribute is not called “Group Tag“, but “OrderID“. The beauty of it is that – for Azure AD joined devices – these two attributes remain inter-linked, meaning that a change in the Group Tag in Autopilot will result in an update of the ODERID value of the corresponding Azure AD device object.

Some PowerShell commands to get you started validating this behaviour would include the following. First, connect to Autopilot via PowerShell to find all device details. If you do not have the PowerShell module installed, you can do so with the following command:


As you can see from the output above, we highlighted the groupTag attribute. Next, connect to Azure AD with PowerShell to find the matching device object and its details to confirm (prove, if you will) the tag is indeed passed from Autopilot to AAD. As you can see, the device’s OrderId attribute should match the groupTag from earlier.


With this information in our back pocket, we can address the requirements described in the beginning of this post, since Azure AD groups support dynamic membership rules (which can be based on the OrderId attribute).

It is very important to come up with a well-thought syntax for this Autopilot tag. As an example, and taking into account many of the (enterprise) customer requirements we have seen in the field, a syntax could look like the following: 

Position	Value	Description
1	A | H	Azure AD joined (or Hybrid AAD joined).
–	 	 
3-5	xxx | yyy	Management authority/department (in larger enterprises, often multiple teams are managing a specific subset of devices.
This can then be used to set Role Based Access Control (RBAC) in Intune
–	 	 
7	G | D	Primary worker type (Generic office user, Developer, …)
–	 	 
9-11	CAN | PRD	Environment (Canary, Production)
–	 	 
13	DT | LT | VM | KS | CD | …	Form factor (Desktop, Laptop, VM, Kiosk, Communications device, …)
– 	 	 
15-16	BE | NL | ..	Country/Entity
…	 	 
We would then typically create so called “building-block” groups, using the below query syntax for dynamic groups as an example. 

To create a group which contains all centrally managed Belgian devices, you could use the following:



Or perhaps you are looking for all centrally managed Canary (pre-prod) devices:  


As you can imagine, the possibilities are endless since this really enables you to simulate a 3-dimensional OU structure, whereby changing the attribute in Autopilot will automatically trigger the affected dynamic group update(s) in the background – much like moving a computer object from one OU to another in an on-premises Active Directory.

We can now, for instance, create two sets of configuration profiles: one for Canaries-devices (CAN) and another one for Production devices (PRD). This allows us to validate settings on a subset of the environment first. Also, take a moment and think a moment what this can mean for managing your Windows Update rings…!

Besides adding a logical DTAP structure to Intune, we can also use this Autopilot tag information to dynamically group devices for Role Based Access Control assignments. For example, support group A is allowed to perform certain device actions (like a remote wipe), but only scoped to a group of Belgian devices, without anybody having to add these devices to the BE group manually. 

As we stated earlier, the possibilities are endless and only limited by your own imagination; provided you have a clearly defined smart syntax for these autopilot tags!