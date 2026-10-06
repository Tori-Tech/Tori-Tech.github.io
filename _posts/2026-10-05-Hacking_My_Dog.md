Today, I hacked my dog.

That's not a sentence you see very often, is it? Well, you're going to see it, and other sentences like it, pretty frequently in this article, because that's exactly what I did. A relative had gifted me a RapidPower smart dog because they thought I'd like it, completely unaware that I would make it the centerpiece of an interesting cybersecurity experiment. 

The dog, which looks like this,

![Picture of a robot dog](dog.webp)

came with a controller populated with buttons. Each button executes a different command. You had your basic left-right-forward controls, (no backwards movement, likely due to the way the wheels rotate; I observed they would lock up if rolled backwards) and more specialized commands that made the dog sit, greet (shake its hand), pretend to swim, "attack", go to sleep, and even do some kung fu.

A cool toy, right? Probably better off in the hands of the children it was made for than in mine, but I had great plans for this otherwise innocent puppy. The box it came in promised voice commands and remote control (without the controller) through an app, but when I navigated to the app store to download it, I discovered that it was so outdated that Google Play refused to allow me to download it.

I initially shrugged it off because playing with the controller was good enough for me; I didn't really like linking all my things together through an app in the first place.

Still, this made me think: What kind of protocol did the robot dog use to connect to the app, and did the app come with sufficiently hardened security? I was determined to find out. I did some research (i.e., I Googled around) and determined that quite a few people have had similar thoughts as me and created tools that let you remotely control the dog from an interactive web dashboard. Not necessarily security-minded, but an interesting premise. However, I'm no script kiddie, so I didn't install these pre-made tools and run them blindly. I was going to reverse engineer this whole process myself.

The first thing I did was download the APK for the app from ``APK.pure`` and imported it into JADX-GUI, my tool of choice. After the program finished decompiling everything, I found myself looking at a highly structured but also highly confusing filetree. This was not my very first time reverse engineering a program, but it was my first time reverse engineering an Android app as I had previously only meddled with simple Python or C++ CrackMes (with very low success rates, I might add, causing me to develop an aversion to reverse engineering in general)

But this seemed easier; doable, even. I did some more Googling around and determined I should first look for evidence of Bluetooth usage, be it library imports or maybe even custom functions. So I used the search feature to search for Bluetooth and got these results:

```com.polidea.rxandroidble2.internal.scan.RxBleInternalScanResultLegacy```

```no.nordicsemi.android.ble.BleManagerCallbacks```


These, sadly, turned out to be standard libraries that were imported by the developers.

So I then decided to search for: ``writeCharacteristic``

And after filtering through many results from the above files, I found this:

``com.zhongrun.robotdog`` 

This was the very service that all the pre-made scripts appeared to control. Now, further investigation on how apps and robot toys communicate via Bluetooth enlightened me on an interesting concept. To control the dog with an app, commands needed to be stored as hex codes, then those hex codes would be converted to bytes, which would then be broadcast to the dog over Bluetooth, specifically, the Bluetooth Low-Energy protocol. This meant that if I wanted to find the hex codes for each command, I'd need to find a part of the source code that was converting hex to bytes.

The simplest way to do that was to literally search for ``hexToBytes``. I ran the command and found this:

```java
   public static byte[] hexToBytes(String str) {
        if (str.length() % 2 != 0) {
            str = DeviceId.CUIDInfo.I_EMPTY + str;
        }
        int length = str.length() / 2;
        byte[] bArr = new byte[length];
        for (int i = 0; i < length; i++) {
            int i2 = i * 2;
            bArr[i] = (byte) Integer.parseInt(str.substring(i2, i2 + 2), 16);
        }
        return bArr;
    }
```

Which was sort of what I was looking for, but not quite. This piece of code seemed to be the portion of the app that actually converted the hex codes to bytes, but the actual codes were nowhere to be seen. I used the JADX-GUI's built in ability to trace where else the function hexToBytes() was being used, and found this:

```java

 private void sendData(String str) {
        RxBleConnection rxBleConnection;
        if (!AppSystem.isBleConnected || (rxBleConnection = this.mRxBleConnection) == null) {
            return;
        }
        rxBleConnection.writeCharacteristic(this.WRITE_UUID, TypeUtils.hexToBytes(str)).subscribe(new Consumer() { // from class: com.zhongrun.robotdog.view.MainActivity$$ExternalSyntheticLambda21
            @Override // io.reactivex.functions.Consumer
            public final void accept(Object obj) throws Exception {
                this.f$0.m148lambda$sendData$22$comzhongrunrobotdogviewMainActivity((byte[]) obj);
            }
        }, new Consumer() { // from class: com.zhongrun.robotdog.view.MainActivity$$ExternalSyntheticLambda22
            @Override // io.reactivex.functions.Consumer
            public final void accept(Object obj) throws Exception {
                this.f$0.m149lambda$sendData$23$comzhongrunrobotdogviewMainActivity((Throwable) obj);
            }
        });
    }

```

This was still not what I was looking for, though. From what I could gather, this was the main function that handles the Bluetooth-to-App communication with the robot dog. First, it checks to make sure the dog is connected, then converts some kind of hex string to bytes so the corresponding command can be broadcast via Bluetooth. The ``writeCharacteristic()`` function broadcasts that data packet through the (supposed) phone to the robot dog.

I spent about an hour tracing functions and checking all the other areas where they were being used. At one point, I embarked on a wild goose chase because of a file name I stumbled upon: ``activity_gou_gou.xml``

"gou" is the Pinyin version of “狗”, which means "dog". “狗狗” could easily be interpreted as "doggy". Upon seeing this, I had very good reason to believe that this file had some information relating directly to the robot dog. I soon discovered I was correct, literally. I discovered that ``.xml`` files are stricly for UI/layout designs, which meant that what I was looking at was code for the layout of some kind of dog object, and would not control the robot dog directly. This made sense, though. The box in which the robot dog came in advertised a sort of 3d-model replica of the robot dog available in the app. 

However, the content inside ``activity_gou_gou.xml`` did point me to a function called ``iv_ctrl`` due to its button-mapping logic. Tracing its source in JADX led me to a resource mapping page, so I searched for ``R.id.iv_ctrl`` and found some more interesting things, namely, two files that looked as though they were responsible for command handling:

```java 
com.zhongrun.robotdog.view.FunctionActivity.initViews()
com.zhongrun.robotdog.view.GouGouActivity.initViews().
```


I went inside ``GouGouActivity`` and discovered this was actually just code for *programming* the dog, which was a feature advertised in the app and possible through the controller. In "programming" mode, you could define a sequence of actions, and at the push of a button, have the robot dog execute all the actions back-to-back. Sadly, this was still not what I was looking for: I needed to find the exact hex codes that correspond to actions within the dog.

So I checked ``FunctionActivity``, where I found something: an array containing actions. My ability to read Chinese is mediocre at best (only up to HSK3 textbook material), so I did not understand all of the commands, but I spotted words like "跳舞” and "坐下“, meaning "dance" and "sit down", respectively. My hypothesis was: Since these are actions that correspond to controls both on the controller and supposedly within the mobile app, the hex codes that are associated with them should be somewhere in this file.

After scrolling through the file for a few minutes, I found a function called ``initCtrlViews()`` and discovered that within it was all the information I needed. The most important part of this code was the switch case statement, which mapped out all the hex codes alongside the actions they triggered."


- Case 0 (Sit Down): "F02A0010D5FFEF"
- Case 1 (Confused/Question): "F02A0013D5FFEC"
- Case 2 (Lie Down): "F02A0016D5FFE9"
- Case 3 (Act Cute / Whine): "F02A0019D5FFE6"
- Case 4 (Shake Hands or Fire): "F02A001CD5FFE3" (Depends on model type, apparently)
- Case 5 (Attack): "F02A001FD5FFE0"
- Case 6 (Show Love / Affection): "F02A0022D5FFDD"
- Case 7 (Urinate): "F02A0025D5FFDA"
- Case 8 (Flip / Somersault): "F02A0028D5FFD7"
- Case 9 (Patrol Mode): "F02A002BD5FFD4"
- Case 10 (Kung Fu): "F02A002ED5FFD1"
- Case 11 (Push-ups): "F02A0031D5FFCE"


Now came the fun part: making the dog respond to these codes. It sounded fun on paper because it would prove to be true "exploitation" of the device, except in practice, actually finding a way to get the dog to receive the commands was a much trickier process. I first tried connecting to the dog with my laptop, using a Python script with the ``bleak`` library, but that didn't work no matter what I tried. As loathe as I was to do it, I had to give up and pivot to a mobile device. 

I ended up installing Universal BLE on an Android phone and connected to the robot dog using that. After some trial and error, I figured out which specific channel to connect to and began entering in some of the hex codes from the list, namely, the one for kung fu, since I liked that one in particular.

The rush of dopamine that flooded my brain as my robot dog immediately began throwing punches was unlike anything else, because it meant that I had hacked my dog.

Ultimately, the main takeaway from this whole experiment is this: The robot dog implements no pairing mechanism with any devices and lacks encryption on its BLE characteristics. Anyone within Bluetooth range can broadcast these exact HEX values to any active RapidPower dog and control it. Although this is quite harmless for a toy, a similar lack of transport-layer security in other IoT devices is a dangerous and perhaps even pervasive issue.
