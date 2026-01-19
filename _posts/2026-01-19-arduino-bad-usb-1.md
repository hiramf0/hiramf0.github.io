## Arduino BadUSB [Part 1]: Initial Setup

### Researching my new BadUSB
*Disclaimer: This article is for educational and hobbyist experimentation only. Only try these techniques on your own devices or systems you have permission to test. I do not condone misuse of BadUSB devices and am not responsible for how this information is used. Please be careful!* <br>

A few months ago, I went to NMSU for a <a href="https://crimsonconnection.nmsu.edu/event/11834123">CTF competition</a> and I received a Bad USB as a prize. At the time, I did not want use the USB with my personal laptop for obvious security reasons. About a month ago, I bought a mini PC and configured Ubuntu and RDP to use as a 'homelab' and avoid potentially hurting my main laptop. Today, I decided to finally start experimenting with this device and, in this blog post, we will look at how I configured this USB.
<br><br>
<img width="257" height="262" alt="image" src="https://github.com/user-attachments/assets/6384afc2-9ef9-45ef-9b4d-01c1ab4e6e4c" />
<br><br>
First, I did some research to verify the model of the USB I had received. After a few minutes, I found the exact same model on <a href="https://www.amazon.com/HiLetgo-Microcontroller-ATMEGA32U4-Development-Keyboard/dp/B07W5K9YHP">Amazon</a> for around $15:<br>
<br>
<img width="1401" height="584" alt="image" src="https://github.com/user-attachments/assets/172ff5ca-1a42-44be-817f-3fd6699f2464" />
<br>
<br>
From this Amazon product, we can gather some key pieces of information:
- The BadUSB runs Arduino.
- It acts as a USB keyboard.
- It is really cheap...

I plugged the USB to my mini PC running Ubuntu and...<br>
<br>
... nothing happened.<br>
<br>
Running `lsusb` further confirms that the USB is connected and recognized as an Arduino device:
```bash
Bus 001 Device 003: ID 2341:8037 Arduino SA Arduino Micro
```

Reading some more reviews on the Amazon page, I realized that it can run `cmd` on a Windows machine and (allegedly) do nothing:<br><br>
<img width="886" height="380" alt="image" src="https://github.com/user-attachments/assets/ee9b7778-5c2e-460a-8ee3-63a148e99f24" />
<br><br>

Let's do our development on Linux and try running scripts on Windows.<br>

### Setting up Arduino IDE
Let's start working with this Arduino device. First, I installed the Arduino IDE to begin development:
```bash
sudo apt install flatpak
flatpak install flathub cc.arduino.arduinoide
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub cc.arduino.arduinoide
echo 'alias arduino="flatpak run cc.arduino.arduinoide"' >> ~/.bashrc
source ~/.bashrc
arduino
```
<br>
Typing 'arduino' in the command line will now open up the Arduino IDE. <br>
Let's setup the IDE to use our desired board and port. <br>

- Tools > Board > Arduino Leonardo
- Tools > Port > /dev/ttyACM0

To be able to start writing Arduino scripts, we need some additional configuration: <br>
- Add our user to the *dialout* group > Log out and log in again <br>
- Give Arduino IDE permission to access serial devices <br>

```bash
sudo usermod -a -G dialout $USER
flatpak override --user --device=all cc.arduino.arduinoide
```

Let's write a sample script to verify we can safely upload our scripts to the USB. <br>
Before uploading the script, open the Serial Monitor to verify the script is working: <br>
Tools > Serial Monitor <br>
```arduino
#include <Keyboard.h>

void setup() {
  Serial.begin(9600);
  while (!Serial) {
    ; // wait for serial on ATmega32U4
  }

  Serial.println("Serial OK. Waiting 5 seconds...");
  delay(5000);

  Serial.println("Starting keyboard...");

  Serial.println("Done typing.");
}

void loop() {}
```

The Serial Monitor should look like this after pressing Upload:<br>
<br>
<img width="1134" height="423" alt="image" src="https://github.com/user-attachments/assets/59f5f158-25f9-411e-bd8d-fd5402548243" />
<br>

### Testing a real script
Now, let's write a simple script to open Notepad on Windows.
```arduino
#include <Keyboard.h>

void setup() {
  // safety delay to remove USB before execution
  delay(5000);

  Keyboard.begin();

  // open Run dialog (WIN + R)
  Keyboard.press(KEY_LEFT_GUI);
  Keyboard.press('r');
  delay(100);
  Keyboard.releaseAll();

  delay(500);

  // type "notepad"
  Keyboard.print("notepad");
  Keyboard.press(KEY_RETURN);
  Keyboard.releaseAll();

  //  wait for notepad to open
  delay(1000);

  // type "hello world!"
  Keyboard.println("hello world!");

  Keyboard.end();
}

void loop() {}
```

I uploaded the script to the USB, disconnected it from my mini PC, and plugged it into my Windows laptop... <br>
<br>
<img width="655" height="478" alt="image" src="https://github.com/user-attachments/assets/de3f68aa-fdaf-4434-9f5e-129ddc91d4c3" />
<br>
It worked! <br>

### Lessons Learned
This USB device acts as a keyboard and can be used for many purposes: to automate common and repetitive IT helpdesk tasks, to compromise an endhost, or to play a prank on an unsuspecting friend. Some other things that I would like to continue exploring are: <br>
- The USB did not work on my mini PC running Ubuntu. Is this because of a security feature or because <a href="https://i.programmerhumor.io/2025/07/8ba29e068c970c5773028649506c83fbac3f0fc403093a4027c8a6e99b911635.jpeg">nothing works on Linux?</a>
- Could I include other files inside this USB and 'drop them' into a victim PC? These files could be images, pre-created scripts, or other media.
- How can USB attacks (like this BadUSB) be mitigated on Windows machines?

I will continue experimenting with this device later. Remember to be careful around unknown USB devices! <br>
<br>
※ This is the end of the article. Thank you for reading!
