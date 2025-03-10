---
layout: post
title:  "Fixing the default switch!"
date:   2025-03-08 20:40:00 +0300
tags: Hyper-V
---
Recently, I had the pleasure of finding and fixing a really annoying error. I personally enjoy debugging errors like these, as they represent a challenge that can’t be solved simply by searching Google. And the feeling after successfully fixing them needs no explanation.

If you have worked with Hyper-V or WSL, you would know that these services create a default switch that can be used for sharing the host's network connection with the VMs. This switch is enabled by default and isn't very flexible. It also recreates itself whenever you open the Hyper-V Switch Manager, making it difficult to disable permanently. Unfortunately, in my case, it somehow stopped appearing.

I did what anyone else would do and searched Google for a solution. Many people had encountered the same error, and a variety of solutions were posted across different forums. However, after trying them all, and going through the endless loop of disabling and enabling Hyper-V, WSL, and Windows Virtualization Platform in different orders, restarting, deleting all switches, and restarting numerous services—it still didn't work.

That's when I decided to take a structured approach. You might wonder, why struggle so much over such a small issue? But my curiosity got the better of me. Besides, I was using some Linux-only software on WSL and had trouble accessing it from the Windows host.

So, I read up on how the Default Switch works. I noted all the Windows services required for its operation and enabled logs for each of them. That’s when I noticed that the Host Networking Service (HNS) was reporting errors on Windows reboot and whenever I opened 'Virtual Switch Management' in Hyper-V. The error wasn’t very descriptive, it simply said:

"'IpICSHlpStartDhcpServer' : '0x80070057'"

If you search for details of the error code you will find that it's hex for "ERROR_INVALID_PARAMETER", which is a generic error code defined in winerror.h .

To investigate further, I used WinDbg and attached it to the svchost process running HNS. I then set breakpoints on every exception. Soon, I identified the exact function that was failing and found the underlying system call responsible for the error. The failing function was 'IpNatHlpRpcClientStartDhcpServer', imported from a DLL named 'IpNatHlpClient.dll'. The system call causing the failure was NdrClientCall3.

I searched for a description of NdrClientCall3 and discovered that it was related to communication with an RPC server. I then retrieved the RPC GUID that the client was trying to connect to and used RPCView (from GitHub) to inspect all RPC servers listening on my machine. I checked whether the GUID 'E64B9AEE-F372-4312-9A14-8F1502B5C8E3' was present. It wasn't.

At this point, I almost understood the problem: A required service for HNS wasn’t running correctly.

A quick Google search revealed that this GUID belongs to the Internet Connection Sharing (SharedAccess) service. I opened services.msc and found that this service was set to Delayed Start instead of Automatic Start, even though it's supposed to be set to Automatic.

I have no idea how or when this change occurred, as the Event Viewer system logs had already been overwritten. Regardless, I changed the startup type to Automatic, and just like that, the default switch started working perfectly. My WSL issues were resolved (at least until I mess it up again somehow).