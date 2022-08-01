# Trash Royale
![](https://img.shields.io/badge/category-misc-blue)
![](https://img.shields.io/badge/solves-53-orange)

## Description
Find the flag, and make sure its wrapped in LITCTF{}

Ugh, I keep forgetting my password for <https://spectacle-deprive4-launder.herokuapp.com/>, so I will just it here.
Username: LexMACS and Password: `!codetigerORZ`

![](https://i.imgflip.com/6njl88.jpg)

## Solution
Logging into the website with the provided credentials, there isn't much on the page. No useful links, and the morse code at the top simply says "HEHEHEHAW".

However, inspecting the source code reveals
```html
        <p style="display: none">
            🔥Welcome early-access testers!!!🔥 Download our game <a href="/3ar1y-4cc35s">here!</a>
        </p>
```

This secret page has a too-obvious link to the flag, which actually resolves to a very famous Rick Astley video. The other two links download two files, trashroyale.exe and godtyger.jpg.

![](https://user-images.githubusercontent.com/62577178/182062737-488e3a87-c967-4b17-84e7-072c1663d397.png)

The exe file looks like just a terrible ripoff of the game Clash Royale, and upon winning, a sequence of obnoxious laughs starts playing. There wasn't much else to do in the app, so I tried to decompile it.

The file icon was the signature PyInstaller logo, so I used [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor) to extract it. Inside the assets folder of the extracted folder was a file named flag.mp3. This audio was the same audio that played at the end of the game, but due to its name, I did some extra investigating. Opening the file in Audacity revealed a waveform that looked clearly like Morse Code, a reference to the morse code on the website from earlier. 

![](https://user-images.githubusercontent.com/62577178/182063604-08f87913-8d34-4d30-9363-dea389daea7f.png)

It was obviously morse code because there were only two distinct sound effect lengths, a short and long one. Decoding the morse code revealed the flag.


Flag: `LITCTF{H3H3H3H4WTHREECROWN}`
