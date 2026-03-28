# Rerouting System Audio on Rooted Android

> These are my personal notes from trying to reroute audio output on a rooted Samsung Galaxy A9+ 5G tablet, which I will refer to by its model name (SM-X216B) from now on. This is not a guide or a recommendation but rather just a record of what I tried, what broke, what I understood (mostly, what I misunderstood), the compromises I had to accept, and what ended up working for me under specific constraints. 
> 
> This repository also contains backups of key files I modified on my system and their altered versions; both of which are only included for reference and personal use. I am not responsible for damage, data loss, or other issues that may result from improper use of these files.

## Context Time

I cannot remember the exact date or month of 2024 when I got a new tablet: an SM-X216B. What I do remember is that as soon as I got it, I immediately rooted the device while keeping the stock ROM since I am an avid Samsung Dex user. Being somewhat of an audiophile, I installed [Viper4Android](https://github.com/AndroidAudioMods/ViPER4Android/releases/tag/v0.6.2) and turned the tablet into my personal media hub using [VLC for Android](https://play.google.com/store/apps/details?id=org.videolan.vlc) for playing music, radio and podcasts alongside [Youtube Revanced](https://revanced.app/) and [Twitch](https://play.google.com/store/apps/details?id=tv.twitch.android.app) for watching streamers while working.

Obviously this being a tablet, I spend a lot more time on both my phone and my laptop than I spend on the tablet. Controlling what to play is simple enough since VLC provides a remote control feature which exposes a page that any device on the same network can access and when I am on my laptop, the tablet is usually right in front of me. So all I needed for an ideal workflow looked pretty straightforward: send the tablet's audio output, post V4A, to my laptop and phone. What follows is a record of my experience trying to achieve that.

## But Many Apps Do This Quite Easily, Right?

If you have ever used the screen share feature on [ZOOM](https://www.zoom.com/), you will know that it allows you to easily share not just the audio but also the visual output of your Android device (I mean, that is the entire point of the feature) after just asking for a couple of permissions. So obviously that is where I started looking.

Now, not to go too deep into Android App Development, Android exposes playback capture via an API called MediaProjection. This API is what apps like ZOOM, Google Meet, etc heavily rely on to be able to read audio/visual output during screenshare.

Unfortunately, there is an intentional caveat implemented in the API. Any app can just stop its content from being captured by adding a window flag called FLAG_SECURE in its code. This was suboptimal, especially because A2DP does not have this caveat and I really did not want to downgrade my system to the point where any app could just make my life miserable by adding one line into its code at any point.

It was clear that nothing could be done on this layer. I needed to go down a few layers.

## I Might Have Gone a Little Too Deep

Alright, so Android's own API won't help me and I couldn't change to a custom ROM because of my dependency on Dex. That basically left very few options for me if I wanted this to be reusable and reliable. I had to think outside the box. I just decided that the box in this case was the entire tablet. Basically if the software is gonna stop me from getting a hold of the audio output then I will just take it from where the output is actually supposed to go: the audio jack.

### The Plan

The idea started simple-ish. Take the output from the audio jack of the tablet, make it look like something a mic would produce, and then just send it back through the same jack it came from. Once the output became a physical input, it would no longer be subject to Android's playback capture restrictions.

### They Say the Devil's in the Details

Alright, let's start from the beginning by looking at the star of the show: the audio jack itself. Now, there are multiple types of audio jacks but since we need one that supports mic input alongside output, that narrows it all down to a TRRS (i.e. Tip-Ring-Ring-Sleeve) jack.

As the name suggests, a TRRS jack contains four contact points, three of which carry signals, while the fourth acts as ground. Now which one does what is actually decided by the device and not the jack itself. Most modern devices follow the CTIA standard. That maps the contacts as:

+ Tip -> Left output
+ Ring 1 -> Right output
+ Ring 2 -> Ground
+ Sleeve -> Mic

So, this means that if we want lossless audio, we are gonna have to properly mix the output from the Tip and Ring 1, make that mixed signal electrically resemble a mic signal, and send that to the sleeve. Sounds simple enough, except, the device needs to detect that something is plugged into its audio port. How do devices do that anyway?

Well, if you just want the output from the port, things are pretty simple: inserting a jack into the port physically pushes a switch inside the port that tells the device that something is plugged in and audio output can be sent through it. Purely mechanical, already solved just by using an actual jack.

The mic, however, is where things get electrical. When a jack is plugged in, the device sends a small voltage signal and expects a completed circuit with a specific resistance for said resistance to be recognized as a mic. Therefore, for the device to accept any form of input, that particular path and resistance needs to be mimicked.

Rewinding a bit, what does a signal produced by a mic look like exactly? A typical mic produces signals in the range of 1 to 10 millivolts and is typically expected through a bias network of around 2.2 kilo-ohms. What we are gonna get as output will be around 1 to 2 Volts with an impedance of like 10 ohms at best. We are gonna have to operate on that output signal like a neurosurgeon just to avoid frying the audio port, let alone get it to accept the input.

### Ok, How Do We Do This?

Bringing something in the range of a couple volts down to millivolts might sound like a job for an op-amp. And you would be right. Except you can't just use any op-amp and call it a day. Thing is, audio signals are extremely sensitive. General purpose op-amps, not so much. What a project like this requires is a low-noise op-amp that does not distort the signal. I decided to go with a TL072 op-amp because honestly, that's the only one I could afford. The only caveat: it's not rail-to-rail. This means that the op-amp cannot operate properly close to the supply voltage limits and cannot handle voltages that swing below ground. Shouldn't be a problem in many cases, right? Except, audio signals are AC, centered around zero and the output needs to be in the order of millivolts (~0.001-0.01V).

This meant that I would have to implement a virtual ground. Basically, the concept involves gaslighting the op-amp into thinking that the "ground" is not actually at 0V, but rather somewhere in the middle of the supply voltage. In my case, I decided to use a 9V DC battery (convenient and sufficiently high for this setup). Now, instead of trying to process a signal that swings around 0V, the op-amp is gonna see the signal swing around 4.5V instead, thus staying comfortably inside the op-amp's practical operating range.

Alright, that's one problem solved. The op-amp can now get us a signal that swings above and below 4.5V. But, how do we get the input signal to the op-amp? Well, you can just short the tip and ring 1 and let the signal from both superpose each other. Effective, but dirty. See, when 2 waves interact, they can introduce interference and imbalance between channels. We need to clean this mixed signal up. This is where high-pass and low-pass filters come in.

Not gonna go into too much theory here but a high-pass filter, as the name suggests, allows signal above a certain, configurable frequency to pass through. In my case, that is around 20Hz, i.e. the lower end of the human hearing range. Similarly, a low-pass filter sets a frequency threshold, under which the signal is allowed to pass. Here, that would be 20kHz.

So, we short the left and right outputs, and have them go through filters that remove low-frequency drift and high-frequency noise. But, this may not ensure that all the DC garbage is taken care of. So, to be extra certain, just throw in a 1uF capacitor that blocks all DC and only allows AC signal to pass through. Now we have a clean signal that we can send to the op-amp.

Now, to deal with the output that the op-amp will produce: the 4.5V bias on which the now attenuated AC signal is riding is not required anymore. So, we get rid of that using another 1uF capacitor. Now we have an AC signal in the order of millivolts which means that we have ensured that the audio port won't get fried. Unfortunately, our problems don't end here.

### The Part Where Everything Goes Wrong

Alright, we have the signal that the device should accept. Now we need to get it into the device. Thing is, as much as I would have preferred it to be the case in this situation, the mic node of the audio port is not just something that will magically read the signal.

For the sake of my personal use-case, I am going to heavily simplify this next part but consider the mic node to be sitting between whatever comes in via the jack and a ~2V voltage source with an internal series resistance of ~2kOhm. Good news first, the node expects an AC signal in the order of a few millivolts riding on the 2V DC and the internal series resistance will just help match the impedance to what the node is expecting.

Now, the bad news: remember how I mentioned that the mic is detected by the node sending a small signal and expecting a complete circuit with nothing but a ~2kOhm resistance in the path? The node won't be able to detect that from where the signal is coming, and so the "mic" never gets detected, the device doesn't accept anything coming at the node and all that I would be left with is a bunch of soldered mess.

So, the first idea was to just add a 2.2kOhm resistor to ground at the same node. Except, now even though the node detects a mic, the signal itself does not survive the process. The node is now shared between the device’s internal bias network, the external resistor, and the incoming signal. In this configuration, the already tiny AC signal gets heavily attenuated and distorted as part of it is effectively shunted to ground. 

Getting the device to recognize a mic and preserving a usable signal at the same time was not going to be easy.

### A Moment of Lucidity

Now, I am not claiming that the problem I just mentioned above is impossible. On quite the contrary, I theorize that you might be able to solve this problem using a JFET or similar high-impedance transistor; but this is the point when I was forced to look back at everything for a second. I mean, all this only to get mono audio at the end (you don't really get the choice with this setup since the mic does not get a left and right). Good quality, low-noise JFETs aren't exactly cheap either; neither do I have the skills to be able to properly prototype something like this with a soldering iron. In short, the cost-to-benefit had stopped making sense and worst case scenario could easily end up being a pile of soldered hardware that just doesn't work.

So where does this leave things? Well, I did crunch the numbers and use LTSpice for both design and simulation. As such, anyone interested will find the schematics in this repository. Personally, while I would like to see that design doing anything useful, I am not going to spend more time on it any further.

TL;DR, trying to grab audio output at hardware level for use ended up in a failure.

## It Ain't Over Just Yet

Ok, the hardware route did not work out. That's a long amount of work down the drain but, whatever. There's still a problem to solve. I decided it was time to go diving into the android audio stack. At this point, I had explored both ends from hardware to high level API. So, the only sensible option left was something in-between. That meant looking into how the audio was routed on my tablet. This led me to looking through audio sinks and if any of them could be intercepted. During that, I came across `remote_submix`.

Let's go on a small history lesson: circa Android 4, before MediaProjection API was even conceived. Prime time, if not a little behind its competitors for Android to introduce features such as Miracast, Screen Capture, and other stuff requiring audio output capturing. Back then, to implement that stuff, a special audio sink was designed to act as a pipe that the output could be routed through. Thus, remote_submix was born.

As of present day, on modern Android, remote_submix has been almost entirely phased out in favor of the MediaProjection API. But it is still just sitting there as a free audio sink. In fact, android actually still allows access to it if you use either root or adb shell, as long as you still use the MediaProjection API as the entry point to interact with (which is an oversimplification of what [scrcpy](https://github.com/genymobile/scrcpy) does to capture audio).

Whatever, there must be a way to wrangle the audio sink. Before figuring that part out, let's start with making remote_submix always available so that I can switch to it at any time.

### So, That Worked a Little Too Well

To make remote_submix always available, all I need to do is modify the audio configuration files inside the `/vendor/etc` directory. Inside I found a file named `r_submix_audio_policy_configuration.xml`:

```
<module name="r_submix" halVersion="2.0">

    <attachedDevices>
	<item>Remote Submix In</item>
    </attachedDevices>

    <mixPorts>
        <mixPort name="r_submix output" role="source">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="48000"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>

        <mixPort name="r_submix input" role="sink">
           <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                    samplingRates="48000"
                    channelMasks="AUDIO_CHANNEL_IN_STEREO"/>
        </mixPort>
   </mixPorts>
.
.
.
```

And just added one line:

```
<module name="r_submix" halVersion="2.0">

    <attachedDevices>
	<item>Remote Submix In</item>
	<item>Remote Submix Out</item>
    </attachedDevices>

    <mixPorts>
        <mixPort name="r_submix output" role="source">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="48000"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>

        <mixPort name="r_submix input" role="sink">
           <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                    samplingRates="48000"
                    channelMasks="AUDIO_CHANNEL_IN_STEREO"/>
        </mixPort>
   </mixPorts>
.
.
.
```

I used Magisk to make this one change into a module so it could be applied to the device during boot and voila, I had made remote_submix always available as an output.

Except, what also happens during boot is that Android decides the priority of every audio output in the configuration. Speakers, headset, etc. take top priority as soon as they are initialized because the hardware driver responsible for them is the only thing available during boot no matter how long it takes to initialize. At least that was the case until I decided to change the configuration so that now, a virtual audio sink; one that will always be able to initialize faster than any audio sink relying on a hardware driver thanks to being virtual, and therefore not dependent on any hardware drivers, is also available during boot. This meant that remote_submix being immediately available during boot, was consistently selected as the active output before hardware-backed sinks had a chance to initialize, resulting in all audio being routed through it by default.

Not an issue since I can probably use Tasker or just a basic shell script to change which audio sink to use as long as it is available and I am personally not the type to complain when my beer is too cold so let's go forward and try to capture audio from it using an API that is more low-level than MediaProjection: AudioRecord to see if this sink can be captured. And, this is what I got:

`permission denied: capture not allowed`

That error is why I couldn't just use AudioRecord on any sink, by the way. It was a bit naive but a part of me was genuinely hoping for the off-chance that I wouldn't encounter those same restrictions with remote_submix. Turns out, I was wrong.

### I Am Really Starting to Get Sick of This

Time to directly tackle the layer causing all of this. While the sink configuration is defined in an XML file, all the policy is enforced via binaries that were written in C++ and compiled. If I were on a Pixel device, I would have gone directly to AOSP long ago. Unfortunately, OneUI source, while derived from AOSP is proprietary. I have no idea what changes are made to the binaries I want to target. But, these were desperate times, and so I had no choice but to try and reverse engineer the binaries.

I decided to use [Ghidra](https://ghidralite.com/) for no particular reason whatsoever but rather just because that's the first name that came up.

Now the important part: which binary to target? I had to be extremely precise here as any incompetence (something I am extremely capable of) could corrupt the binary and brick the device. On Android, a major binary that manages audio routing, policy decisions, stream types and directly interacts with AudioFlinger is `/system/lib64/libaudiopolicyservice.so`. So, I decided to pull it and take a look.

Now, I wasn't exactly going in completely blind. A basic way to analyze any binary, is to look for strings that the decompiled code is printing and trace the logic around it. Luckily in my case, I had a very precise target: the string "permission denied: capture not allowed". That is when I found this block:

```
                        switch(local_434) {
                        case 0:
                        case 3:
                          break;
                        case 1:
code_r0x00150ad0:
                          if ((uVar13 & 1) == 0) {
                            pcVar17 = "%s permission denied: capture not allowed";
LAB_00150c90:
                            __android_log_print(6,"AudioPolicyIntefaceImpl",pcVar17,
                                                "getInputForAttr");
                            goto LAB_00150ca8;
                          }
                          break;
                        case 2:
                          uVar19 = android::modifyAudioRoutingAllowed
                                             ((AttributionSourceState *)param_6);
                          if (((uVar19 & 1) == 0) && ((bVar10 & uStack_188._6_1_ & 1) == 0)) {
                            pcVar17 = "%s permission denied for remote submix capture";
                            goto LAB_00150c90;
                          }
                          break;
                        case 4:
                          if ((bVar10 & uStack_188._6_1_ & 1) == 0) {
                            uVar19 = android::EDMNativeHelper::isPackageInAvrWhitelist
                                               (*(int *)(param_6 + 0xc));
                            if ((uVar19 & 1) == 0) goto code_r0x00150ad0;
                            __android_log_print(4,"AudioPolicyIntefaceImpl",
                                                "%s() captureAudioOutputAllowed for KNOX UID : %d",
                                                "getInputForAttr",*(undefined4 *)(param_6 + 0xc));
                          }
                          break;
                        default:
                          auVar23 = __android_log_assert
                                              (0,"AudioPolicyIntefaceImpl",
                                               "%s encountered an invalid input type %d",
                                               "getInputForAttr");
                          if (auVar23._0_8_[0x18] != (string)0x0) {
                            std::string::string(this,auVar23._0_8_);
                            return;
                          }
                          *(undefined8 *)this = 0;
                          *(undefined8 *)(this + 8) = 0;
                          *(undefined8 *)(this + 0x10) = 0;
                          __n = strlen(auVar23._8_8_);
                          if (0xffffffffffffffef < __n) {
                    /* WARNING: Subroutine does not return */
                            std::__basic_string_common<true>::__throw_length_error();
                          }
                          if (__n < 0x17) {
                            __dest = this + 1;
                            *this = SUB41((int)__n << 1,0);
                            if (__n == 0) goto LAB_00150fec;
                          }
                          else {
                            __dest = operator.new((__n | 0xf) + 1);
                            *(size_t *)(this + 8) = __n;
                            *(string **)(this + 0x10) = __dest;
                            *(size_t *)this = (__n | 0xf) + 2;
                          }
                          memcpy(__dest,auVar23._8_8_,__n);
LAB_00150fec:
                          __dest[__n] = (string)0x0;
                          return;
                        }
```

This looked promising at first, but what really sold me on this being the point of enforcement was `case 4`. Knox is Samsung's enterprise security framework. A branch specifically whitelisting apps associated with Knox meant that the Samsung developers faced a problem not quite unlike the one I was facing, except they had the source and found this to be the most optimal spot to implement a whitelist at. Now, the amount of code in each branch is certainly not enough to do everything from denying the request to handling the state stuff. And that is where I believe the line `goto LAB_00150ca8` comes in. It redirects the execution to an instruction stored at the address `LAB_00150ca8` but only in the branches that we know are denying the request. So, if I patch my specific case to break before that goto instruction is called, I should be golden.

But something was off. There is a specific case for remote_submix and even though I am using it, that is not the branch I am hitting. At that point, though I just decided to patch both branches and call it a day. That meant changing this instruction in both branches:

```
00150c84 88 06 00 37        tbnz       w8,#0x0,switchD_00150ac8::caseD_0
```

to this:

```
00150c84 88 06 00 37        b       switchD_00150ac8::caseD_0
```

Then I exported the patched libraries, added them to the module I had made for always enabling remote_submix, and tested AudioRecord again.

**And it worked!** AudioRecord started running without errors and printed PCM data that actually looked valid!

### So, What Exactly Did I Have At This Point?

Well for starters, what I did not have anymore was the time I spent on this entire ordeal spanning across 2 years. But I didn't just end up with a way to route audio from my tablet to literally any device I wanted. I also gained the ability to capture it, or modify it post everything from the source app to V4A. Of course I would still have to build the app to get the audio and stream it to my other devices myself but that felt like a victory lap compared to everything that came before it.

However, there was one caveat: if you tried to capture audio from a speaker on an Android device, you would get `permission denied: capture not allowed`. I say that in case it wasn't obvious that the `case 1` branch is a generic route for disallowing any form of disapproved output capture, including speakers, headphones, etc. In my quest to reroute audio without restrictions, I had scorched earth and opened the Pandora's Box of security concerns. Honestly though, I didn't even care as this was something you would have to write custom code for if you wanted to attack it and as far as I know I am not on someone's hit list... yet...

So, I won. This is the part where I get to live happily ever after, right? Except...

## I Got Curious

Ok, that switch-case statement was clearly modified by Samsung and not created by them. I had to know if I could find that switch-case in AOSP just because I really wanted to know what any of those variables meant.

So, I got my hands on the `av` (aka. the Audio-Video) repository that handles all multimedia stuff in AOSP. I did some digging to find that my tablet was on android 14, security patch February 2025 which corresponded to the `android-platform-14.0.0_r24` branch (which conveniently happened to be the last android 14 platform release by the way). After that I just searched for the same "permission denied: capture not allowed" string (God bless `ripgrep`) and found this inside `av/services/audiopolicy/service/AudioPolicyInterfaceImpl.cpp`:

```
        if (status == NO_ERROR) {
            // enforce permission (if any) required for each type of input
            switch (inputType) {
            case AudioPolicyInterface::API_INPUT_MIX_PUBLIC_CAPTURE_PLAYBACK:
                // this use case has been validated in audio service with a MediaProjection token,
                // and doesn't rely on regular permissions
            case AudioPolicyInterface::API_INPUT_LEGACY:
                break;
            case AudioPolicyInterface::API_INPUT_TELEPHONY_RX:
                if ((attr.flags & AUDIO_FLAG_CALL_REDIRECTION) != 0
                        && canInterceptCallAudio) {
                    break;
                }
                // FIXME: use the same permission as for remote submix for now.
                FALLTHROUGH_INTENDED;
            case AudioPolicyInterface::API_INPUT_MIX_CAPTURE:
                if (!canCaptureOutput) {
                    ALOGE("%s permission denied: capture not allowed", __func__);
                    status = PERMISSION_DENIED;
                }
                break;
            case AudioPolicyInterface::API_INPUT_MIX_EXT_POLICY_REROUTE:
                if (!(modifyAudioRoutingAllowed(attributionSource)
                        || ((attr.flags & AUDIO_FLAG_CALL_REDIRECTION) != 0
                            && canInterceptCallAudio))) {
                    ALOGE("%s permission denied for remote submix capture", __func__);
                    status = PERMISSION_DENIED;
                }
                break;
            case AudioPolicyInterface::API_INPUT_INVALID:
            default:
                LOG_ALWAYS_FATAL("%s encountered an invalid input type %d",
                        __func__, (int)inputType);
            }
        }
```

I don't know why that `FIXME` still exists by the way. I suppose someone just forgot to remove it after implementing different permission for remote submix? Either way, now we know what each case is:

case 0 -> API_INPUT_LEGACY
case 1 -> API_INPUT_MIX_CAPTURE
case 2 -> API_INPUT_MIX_EXT_POLICY_REROUTE
case 3 -> API_INPUT_MIX_PUBLIC_CAPTURE_PLAYBACK

The rest of the cases were probably just decompiled as if-else statements inside these ones. Happens all the time!

### But Why Am I Hitting the Wrong Case?

Things weren't adding up:

- I *was* using remote_submix
- But I wasn't hitting the remote_submix permission branch

So, I decided to look deeper by tracing `inputType` to where it was being assigned. I found what I was looking for in `av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp`:

```
status_t AudioPolicyManager::getInputForAttr(const audio_attributes_t *attr,
                                             audio_io_handle_t *input,
                                             audio_unique_id_t riid,
                                             audio_session_t session,
                                             const AttributionSourceState& attributionSource,
                                             audio_config_base_t *config,
                                             audio_input_flags_t flags,
                                             audio_port_handle_t *selectedDeviceId,
                                             input_type_t *inputType,
                                             audio_port_handle_t *portId)
{
.
.
.
    *input = AUDIO_IO_HANDLE_NONE;
    *inputType = API_INPUT_INVALID;

    if (attributes.source == AUDIO_SOURCE_REMOTE_SUBMIX &&
            extractAddressFromAudioAttributes(attributes).has_value()) {
        status = mPolicyMixes.getInputMixForAttr(attributes, &policyMix);
        if (status != NO_ERROR) {
            ALOGW("%s could not find input mix for attr %s",
                    __func__, toString(attributes).c_str());
            goto error;
        }
        device = mAvailableInputDevices.getDevice(AUDIO_DEVICE_IN_REMOTE_SUBMIX,
                                                  String8(attr->tags + strlen("addr=")),
                                                  AUDIO_FORMAT_DEFAULT);
        if (device == nullptr) {
            ALOGW("%s could not find in Remote Submix device for source %d, tags %s",
                    __func__, attributes.source, attributes.tags);
            status = BAD_VALUE;
            goto error;
        }

        if (is_mix_loopback_render(policyMix->mRouteFlags)) {
            *inputType = API_INPUT_MIX_PUBLIC_CAPTURE_PLAYBACK;
        } else {
            *inputType = API_INPUT_MIX_EXT_POLICY_REROUTE;
        }
    } else {
        if (explicitRoutingDevice != nullptr) {
            device = explicitRoutingDevice;
        } else {
            // Prevent from storing invalid requested device id in clients
            requestedDeviceId = AUDIO_PORT_HANDLE_NONE;
            device = mEngine->getInputDeviceForAttributes(attributes, uid, session, &policyMix);
            ALOGV_IF(device != nullptr, "%s found device type is 0x%X",
                __FUNCTION__, device->type());
        }
        if (device == nullptr) {
            ALOGW("getInputForAttr() could not find device for source %d", attributes.source);
            status = BAD_VALUE;
            goto error;
        }
        if (device->type() == AUDIO_DEVICE_IN_ECHO_REFERENCE) {
            *inputType = API_INPUT_MIX_CAPTURE;
        } else if (policyMix) {
            ALOG_ASSERT(policyMix->mMixType == MIX_TYPE_RECORDERS, "Invalid Mix Type");
            // there is an external policy, but this input is attached to a mix of recorders,
            // meaning it receives audio injected into the framework, so the recorder doesn't
            // know about it and is therefore considered "legacy"
            *inputType = API_INPUT_LEGACY;
        } else if (audio_is_remote_submix_device(device->type())) {
            *inputType = API_INPUT_MIX_CAPTURE;
        } else if (device->type() == AUDIO_DEVICE_IN_TELEPHONY_RX) {
            *inputType = API_INPUT_TELEPHONY_RX;
        } else {
            *inputType = API_INPUT_LEGACY;
        }
.
.
.
}
```

So, if the source is remote_submix, the if-statement also checks for the address to where the audio is going; which, in retrospect, makes a lot of sense. See, remote_submix is like a pipe that takes the output from AudioFlinger and sends it somewhere. If that "somewhere" isn't declared, android assumes that the other end of this pipe is closed, which effectively turns the pipe into an end-point, hence hitting this block:

```
        } else if (audio_is_remote_submix_device(device->type())) {
            *inputType = API_INPUT_MIX_CAPTURE;
```

In simpler terms, Android treats remote_submix differently depending on whether it knows where the audio is going.

### Can I Make It Hit the Block I Want?

The problem is that making remote_submix always available since boot means I don't get to provide an address. Besides, from this block, it also looks like the address is tied to the MediaProjection API:

```
        if (is_mix_loopback_render(policyMix->mRouteFlags)) {
            *inputType = API_INPUT_MIX_PUBLIC_CAPTURE_PLAYBACK;
        } else {
            *inputType = API_INPUT_MIX_EXT_POLICY_REROUTE;
        }
```

since API_INPUT_MIX_PUBLIC_CAPTURE_PLAYBACK corresponding to MediaProjection API is the only valid route here as we know from default `/system/lib64/libaudiopolicyservice.so` that API_INPUT_MIX_EXT_POLICY_REROUTE leads to permission denial.

So, instead of dealing with the address handling, I decided to just patch this library similar to how I had patched the other library such that the `source = remote_submix && address == NULL` path assigns `*inputType = API_INPUT_MIX_EXT_POLICY_REROUTE`. This way I can leave the API_INPUT_MIX_CAPTURE branch untouched, thus significantly reducing the blast radius to just the remote_submix pipe rather than weakening capture restrictions across the entire system. Like using TNT instead of a nuke to get the job done!

### Here We Go Again, One Last Time

This time I pulled `/system/lib64/libaudiopolicymanagerdefault.so` from my tablet and imported it into Ghidra. Unfortunately, this time there wasn't a clean switch-case or a reliable string to find as branches and code can get rearranged. Luckily, the name of the function "AudioPolicyManager::getInputForAttr" survived and I was able to at least locate that.

That trimmed it down to about 550 lines of decompiled code that I had to go through. Next I decided to see if I could find which variable in the decompiled code actually was inputType. Oh wait, Ghidra already did that for me:

```
/* android::AudioPolicyManager::getInputForAttr(audio_attributes_t const*, int*, int,
   audio_session_t, android::content::AttributionSourceState const&, audio_config_base*,
   audio_input_flags_t, int*, android::AudioPolicyInterface::input_type_t*, int*) */

ulong __thiscall
android::AudioPolicyManager::getInputForAttr
          (AudioPolicyManager *this,audio_attributes_t *param_1,int *param_2,undefined4 param_3,
          undefined4 param_5,long param_6,int *param_7,uint param_8,uint *param_9,int *param_10,
          int *param_11)

{
.
.
.
}
```

Next I needed to find everywhere `param_10` was being assigned in the function. We can clearly see from the source that the address of `inputType` is passed into the function as this allows the function to write directly into the variable rather than create a local copy. Then I filtered them as I was looking for only the assignments that were assigning `param_10` to 1. That left me with this 2 lines:

```
.
.
.
001c9f24 28 00 80 52        mov        w8,#0x1
.
.
.
001ca228 28 00 80 52        mov        w8,#0x1
.
.
.
```

Now, truthfully at this point I just decided to change it to this:

```
.
.
.
001c9f24 28 00 80 52        mov        w8,#0x2
.
.
.
001ca228 28 00 80 52        mov        w8,#0x1
.
.
.
```

And when that didn't work, I changed it to this:

```
.
.
.
001c9f24 28 00 80 52        mov        w8,#0x1
.
.
.
001ca228 28 00 80 52        mov        w8,#0x2
.
.
.
```

Which worked perfectly alongside the new `/system/lib64/libaudiopolicyservice.so` that only had its API_INPUT_MIX_EXT_POLICY_REROUTE branch altered!

## Well, I Did It!

No fakeouts this time. I am actually done. Sort of. As of writing this I am still working on the app that will actually stream the audio from remote_submix to whatever device I want but that's a different thing entirely from this stuff so I am not gonna talk about it here (if you want to know more about that, you can check it out [here](https://github.com/Tahskajuha/remote-submix-streamer)).

At this point, what I have is a controlled and reliable way to capture and reroute system audio post-processing without relying on fragile APIs or hardware hacks. It isn't scalable to every device out there or even safe enough to do so but it is surprisingly elegant for personal use. Let me explain:

Remember the problem where remote_submix always managed to snatch top-priority from every other audio sink? I wasn't going to ever deal with this problem but now that it is not treated like a generic audio sink, it doesn't compete with audio sinks that have `inputType = API_INPUT_MIX_CAPTURE`! Right now, all audio is routed to remote_submix only if there is anything listening on the other end of the pipe; otherwise remote_submix is ignored and audio is routed normally!

I am not gonna end this by pretending it was actually about understanding android better. Nope, A2DP is still as horrible as when I started this journey, and I still want post V4A audio streamed to my phone without actually rooting it. I cannot wait to finish the streaming + recieving logic and finally be free of the curse that is A2DP!

I'm off!
