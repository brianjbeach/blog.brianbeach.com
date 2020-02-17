---
layout: post
title: Pearson OnVUE Broke Mac Mission Control
date: '2020-02-17'
author: Brian
tags: 
- Mac
---


# Pearson OnVUE Broke Mac Mission Control

Pearson has started offering Online Proctored exams. You download the OnVUE application and take the exam from the comfort of your home. This sounds great, but the actual experience was poor. Here is how to fix Mission Control and App Expose after taking an exam.

I took an exam last week. I won’t mention which because I have not passed it yet. I have never failed a certification exam, but I am superstitions and never talk about an exam until I clear it. I uploaded a few pictures of my work area and downloaded the OnVUE app. It asks you to close any running applications before the exam starts, and a proctor watches you while you take the exam.

The proctor actually starts the exam for you. The first time they did, the screen went black and the exam never started. I assume this is when things went wrong. The proctor came on (over my laptop speakers) and asked me to reboot my machine. The OnVUE app had taken over the screen at this point, and I could not get out of the app to restart. I was told to hold the power button to force a reboot.

I started the exam again and everything is going smooth for a few minutes. Then about five question in, OnVUE detected that Amazon WorkDocs had opened a window and ended the exam. I assume a notification from WorkDocs Drive popped up to tell me someone had shared a doc with me. Note that WorkDocs is a background application. OnVUE had not detected that it was running, and did not ask me to stop it before stating the exam.

OK, I accept that OnVUE has to assume I was cheating and end the exam. I talked to support, and they offered to reschedule the exam for free. Note that I plan to go to a test center next time. However, I noticed shortly after that gestures to access Mission Control were not working. App Expose was not working either, but I never use that. I confirmed the gesture was enabled. I tried hot corners. I tried Ctrl-UpArrow. None of them worked.

I opened another case, and the support guy told me that “OnVUE does not make any changes to your local computer”. Maybe that is true, but it seems suspect that this happened right after the exam. It also seems very reasonable that OnVUE would block app switching, Mission Control, etc. while you are in an exam. However, note that support denied it, and I could be wrong.

So, this sent me on an adventure to figure out how to enable Mission Control. After Googling a bit, I found the answer. You can, and I assume that OnVUE did, disable Mission Control using Managed Client for Mac OS X (MCX). I found that it was enabled by running:

```
$ defaults read com.apple.dock mcx-expose-disabled
```

Which returns a **1** indicating **TRUE**. To enable it, simply delete the 
value and restart the **Dock** service by running:

```
$ defaults delete com.apple.dock mcx-expose-disabled
$ killall Dock
```

Once that was done, Mission Control (and App Expose) both worked again. I cannot prove that OnVUE changed that setting, but I almost certain it did.