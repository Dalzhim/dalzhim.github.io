---
tags: c++
title: Trust your tooling
---
When faced with weird results coming from everyday tools, it can be very tempting to discard those results in disbelief by calling it a bug. The fact that those weird results are rare occurrences, that they may be not reproducible in the developer's environment, and most importantly, that the developer cannot understand how to explain them, are strong incentives to discard those reports. And yet, it is very important to fight the urge to mistrust your tools because in most occurrences, they are absolutely right.

Very recently, I helped a coworker with this kind of subtle problem. His team had a crash log coming in from customers showing that our plugin for some other software was crashing the application while calling the boost libraries. In order to illustrate this situation, I've reproduced a simplified crash log where the simplified stack frames read like the following :

```cpp
4 boost::system::some_function() //Path to system-wide: libboost_system.so
3 MyPlugin::initialization() //Path to: MyPlugin.so
2 MyPlugin::onload() //Path to: MyPlugin.so
1 ThirdPartyApp::pluginManager() //Path to: ThirdPartyApp
0 ThirdPartyApp::main() //Path to: ThirdPartyApp
```

Now the important detail about that stack trace is the call from frame 3 to frame 4. What proved challenging was that the team could not explain how this situation could be possible. `MyPlugin.so` statically links to the boost system library and they believed the path to the system-wide dynamic library was a bug of the crash reporter or the symbolicator. And so, they were looking for new hypothesis to work with. That's where I stopped them, before any more work was done. We needed to make sure that they were right about that conclusion before digging elsewhere.

Now fortunately for us, the team had one environment where they successfully reproduced the problem. The plugin worked fine everywhere else. This meant we could experiment with the problematic environment and perform tests in order to identify the critical difference which led to the problem. Our first step was to make sure that `libboost_system.so` does exist at the path reported by the crash log. And it did. On both a stable environment, and the problematic environment.

We then set out to prove that the static link did work by making sure the boost symbol reported by the crash log does exist within our binary (`MyPlugin.so`). To get that done, we used the following command line: `nm MyPlugin.so | c++filt | grep boost::system::some_function`. The `nm` utility lists all of the symbol names exposed by a binary. This means a list of every exported symbol. Then, the output is piped through `c++filt` which demangles the symbol names and makes them easier to read by humans. Finally, `grep` filters the output looking for our search string. Surely enough, we found the symbol we were looking for.

Up to this point, we have proved that there are two different locations where the code exists. But we have yet to prove which one really gets called at runtime on both environments. Some might argue that the crash log proves what is happening, but this is precisely the hypothesis we are trying to challenge.

One way to prove what is happening is to hook up a debugger and set a breakpoint. But there is another way we can achieve the same result without installing any developer tools on the problematic environment: activating some debugging features of the dynamic linker.

Various logging and lookup features of the dynamic linker can be configured with environment variables. In this particular case, it is the symbol bindings that are of interest. Linux's dynamic linker `ld` produces the logs with the `LD_DEBUG=bindings` environment variable. On macOS, look for `dyld`'s `DYLD_PRINT_BINDINGS=1` to achieve the same result.

Here's a trimmed down exerpt of that logging on macOS:

```cpp
dyld: bind: ThirdPartyApp:0x10B93C000 = libdyld.dylib:dyld_stub_binder, *0x10B93C000 = 0x7FFFB1483168
dyld: lazy bind: ThirdPartyApp:0x10B93C010 = libdyld.dylib:_dlopen, *0x10B93C010 = 0x7FFFB14847F7
dyld: forced lazy bind: ThirdPartyApp:0x10B93C080 = libboost_system.dylib:some_function(), *0x10B93C080 = 0x10B972310
dyld: bind: MyPlugin.dylib:0x10B973000 = libc++.1.dylib:std::__1::cout, *0x10B973000 = 0x7FFFBA136660
dyld: bind: MyPlugin.dylib:0x10B973010 = libc++abi.dylib:___gxx_personality_v0, *0x10B973010 = 0x7FFFB0091FC0
dyld: bind: MyPlugin.dylib:0x10B973018 = libdyld.dylib:dyld_stub_binder, *0x10B973018 = 0x7FFFB1483168
dyld: forced lazy bind: MyPlugin.dylib:0x10B973028 = libunwind.dylib:__Unwind_Resume, *0x10B973028 = 0x7FFFB16CEE8E
[…]
dyld: forced lazy bind: MyPlugin.dylib:0x10B973030 = libboost_system.dylib:some_function(), *0x10B973030 = 0x10B972310
[…]
dyld: forced lazy bind: MyPlugin.dylib:0x10B9730A8 = libc++abi.dylib:std::terminate(), *0x10B9730A8 = 0x7FFFB0091D90
dyld: forced lazy bind: MyPlugin.dylib:0x10B9730B0 = libc++abi.dylib:___cxa_begin_catch, *0x10B9730B0 = 0x7FFFB00917E1
dyld: forced lazy bind: MyPlugin.dylib:0x10B9730B8 = libc++abi.dylib:___cxa_end_catch, *0x10B9730B8 = 0x7FFFB0091855
dyld: forced lazy bind: MyPlugin.dylib:0x10B9730C0 = libsystem_platform.dylib:_memset, *0x10B9730C0 = 0x7FFFB169B34E
dyld: forced lazy bind: MyPlugin.dylib:0x10B9730C8 = libsystem_c.dylib:_strlen, *0x10B9730C8 = 0x7FFFB14BDB40
dyld: lazy bind: ThirdPartyApp:0x10B93C018 = libdyld.dylib:_dlsym, *0x10B93C018 = 0x7FFFB1484888
```

I have isolated one line in the middle of the output which shows what we were looking for. There is a symbol within `MyPlugin.dylib` (the macOS equivalent of `MyPlugin.so`) being bound to the system-wide boost dynamic library. So that's our proof that the tools are right. The crashlog is not misleading and rather than searching for a solution blindly while ignoring the best lead, the investigation can get back on track.

Now, my point with this article is that one should never dismiss some results with mere assumptions. Proofs can go a long way saving precious time. But I understand some of you might still be puzzled about how a function call into a static library could get rebound. You have two options. The first option is to read [this great article](http://hacksoflife.blogspot.ca/2012/12/static-libraries-and-plugins-global-pain.html) on the technical details explaining the problem. The second option is to read the following paragraph which summarizes it much more briefly. In both cases, you may want to toy around with the sample program I made that reproduces the issue.

Now if you are still reading, it means you want the spoiler version. Basically, the problem is that `MyPlugin.so` didn't properly hide its internal symbols: everything was global. So even though `Boost::System` was statically linked to `MyPlugin.so`, the dynamic linker is still allowed to rebind the call to the statically linked `Boost` to the other `Boost` instance that has been previously linked in by `ThirdPartyApp`. Yes, exactly. Maybe you missed it the first time you read the exerpt a few paragraphs back, but there was a indeed a line that was logged showing `ThirdPartyApp` linking the same symbol name from `Boost::System` as what `MyPlugin.so` is going to use. Now this problem is easy to reproduce on Linux system, but there is an important additional detail that concerns the Mac. Mac uses a 2-level namespace system meant to prevent this type of accidental binding from happening. You will find an additional linker flag meant to deactivate this protection in the `CMakeList.txt` of my sample program on github.
