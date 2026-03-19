# Rerouting System Audio on Rooted Android

> These are my personal notes from trying to reroute audio output on a rooted Samsung Galaxy A9+ 5G tablet, which I will refer to by its model name (SM-X216B) from now on. This is not a guide or a recommendation but rather just a record of what I tried, what broke, what I understood (mostly, what I misunderstood), the compromises I had to accept, and what ended up working for me under specific constraints. 
> 
> This repository also contains backups of key files I modified on my system and their altered versions; both of which are only included for reference and personal use. I am not responsible for damage, data loss, or other issues that may result from improper use of these files.

## Context Time

I cannot remember the exact date or month of 2024 when I got a new tablet: an SM-X216B. What I do remember is that as soon as I got it, I immediately rooted the device while keeping the stock ROM since I am an avid Samsung Dex user. Being somewhat of an audiophile, I installed [Viper4Android](https://github.com/AndroidAudioMods/ViPER4Android/releases/tag/v0.6.2) and turned the tablet into my personal media hub using [VLC for Android](https://play.google.com/store/apps/details?id=org.videolan.vlc) for playing music, radio and podcasts alongside [Youtube Revanced](https://revanced.app/) and [Twitch](https://play.google.com/store/apps/details?id=tv.twitch.android.app) for watching streamers while working.

Obviously this being a tablet, I spend a lot more time on both my phone and my laptop than I spend on the tablet. Controlling what to play is simple enough since VLC provides a remote control feature which exposes a page that any device on the same network can access and when I am on my laptop, the tablet is usually right in front of me. So all I needed for an ideal workflow looked pretty straightforward: send the tablet's audio output, post V4A, to my laptop and phone. What follows is a record of my experience trying to achieve that.

## But Many Apps Do This Quite Easily, Right?

If you have ever used the screen share feature on [ZOOM](https://www.zoom.com/), you will know that it allows you to easily share not just the audio but also the visual output of your Android device (I mean, that is the entire point of the feature) after just asking for a couple of permissions. So obviously that is where I started looking.

Now, not to go too deep into Android App Development, basically for any app, Android requires that a manifest.xml file be defined which basically contains everything that the app wants to do on the device. This is done by declaring the specific system APIs the app intends to use inside that file. When the app is run for the first time, the system reads the manifest and generates permissions depending on what it sees in the file. For reading audio/visual output of a device, Android exposes the MediaProjection API. Mentioning this API in the manifest and the user allowing the corresponding permission at runtime is the only way apps like ZOOM, Google Meet, etc. are able to read audio/visual output during screenshare.

Unfortunately, there is an intentional caveat implemented in the API. Any app can just stop its content from being captured by adding a window flag called FLAG_SECURE in its code. This was suboptimal, especially because A2DP does not have this caveat and I really did not want to downgrade my system to the point where any app could just make my life miserable by adding one line into its code at any point.

It was clear that nothing could be done on this layer. I needed to go down a few layers.

## I Might have Gone a Little Too Deep

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
