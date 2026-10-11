---
aliases: ["/blog/castors-two-paths/"]
title: "CastorsCTF | Two Paths"
date: 2020-06-01T02:37:56+05:30
slug: ""
description: "Writeup for two paths in castors ctf."
keywords: ["CastorsCTF", "2020", "Cyber Security", "Writeup"]
draft: false
tags: ["writeup"]
math: false
toc: true
---

## Description
```
The flag is somewhere in these woods, but which path should you take?
```
#### [two-paths.png](https://github.com/xatuke/bin/raw/master/two-paths.png)

### Cracking
- We have a classic steganography challenge here but as this is from the crypto category, there must be that also along the way in it.
- I use the `strings` command on the image to see if they've hidden some readable data in this image and I find that there's some binary data at the end of the image.
```
01101000 01110100 01110100 01110000 01110011 00111010 00101111 00101111 01100111 01101111 00101110 01100001 01110111 01110011 00101111 00110010 01111010 01110101 01000011 01000110 01000011 01110000
```
- I use perl to convert this to ascii with the following command:
```bash
echo "01101000011101000111010001110000011100110011101000101111001011110110011101101111001011100110000101110111011100110010111100110010011110100111010101000011010001100100001101110000" | perl -lpe '$_=pack"B*",$_'
```
- Output:
```
https://go.aws/2zuCFCp
```
- The link has expired as the CTF ended but [here's](https://github.com/xatuke/bin/blob/master/decode_this_two_path.html) the html page that I got after going to the link.
- As the challenge name says two paths and the above link only contains some random emojis, I ran [`stegsolve`](https://github.com/zardus/ctf-tools/tree/master/stegsolve) on the image and I found another link leading to another image of a chat.
![castorsCTF](https://i.imgur.com/JgEW7I6.png)

    [https://go.aws/2X1R6H7](https://i.imgur.com/EHtidSw.png)

- This image has a chat between two people who are talking about Alex's party or something, Messages from one person are in plain text but from the other person are encrypted with emojis.
- Use the chat to figure out the mappings of the emojis to ascii characters and use that to get the flag.
- My friend [Diogo](https://github.com/diogoscf), [Clash](https://github.com/aadibajpai) and [Halcyon](https://github.com/Uzay-G) helped in mapping out the emojis and Diogo wrote the python script to get the flag.

### Script
```
analysis = {
  "♈": "c",
  "♓": "o",
  "♒": "n",
  "🌀": "g",
  "🔁": "r",
  "♉": "a",
  "❌": "t",
  "🈲": "u",
  "♏": "l",
  "⏺": "i",
  "♊": "s",
  "💯": "f",
  "🔟": "y",
  "✖": "e",
  "⛎": "d",
  "⏫": "h",
  "➗": "v",
  "♑": "p",
  "🚺": "w",
  "➿": "j",
  "Ⓜ": "m",
  "♌": "b",
  "🔴": "x",
  "♐": "q",
  "🆔": "k",
  "📶": "z"
}

emoji_encoded = """♈♓♒🌀🔁♉❌🈲♏♉❌⏺♓♒♊!_⏺💯_🔟♓🈲_♈♉♒_🔁✖♉⛎_❌⏫⏺♊,_❌⏫✖♒_🔟♓🈲_⏫♉➗✖_♊♓♏➗✖⛎_❌⏫✖_♈⏺♑⏫✖🔁!_🚺✖_➿🈲♊❌_⏫♓♑✖_🔟♓🈲_💯♓🈲♒⛎_♉_Ⓜ♓🔁✖_✖💯💯⏺♈⏺✖♒❌_🚺♉🔟_💯♓🔁_⛎✖♈⏺♑⏫✖🔁⏺♒🌀_❌⏫♉♒_🌀♓⏺♒🌀_♓♒✖_♌🔟_♓♒✖_♓🔁_✖♏♊✖_🔟♓🈲_🚺♓♒'❌_🌀✖❌_❌⏫🔁♓🈲🌀⏫_❌⏫✖_♒✖🔴❌_♑♉🔁❌_➗✖🔁🔟_♐🈲⏺♈🆔♏🔟._Ⓜ♉🔟♌✖_❌🔁🔟_♌✖⏺♒🌀_♉_♏⏺❌❌♏✖_Ⓜ♓🔁✖_♏♉📶🔟!
♈💯♒_🆔⏺🔴♑📶♌♒_🌀🔴♌🔁♓🆔⏫⏫🔴⏫_Ⓜ🈲♊{⛎🈲⏺⏫🈲♈➗_🈲⏫🆔🆔}♊⛎➿⏫💯♓_🆔🈲{🔴🈲⏫♒♑♒Ⓜ}⛎_📶🔁♒♑♒♒✖🔟_💯♑_♈🆔♒🔴🔁🔁✖{♓📶♏_♈❌🌀❌🔁⏫♒♈}🔴♊_{♓💯♉💯⛎➿♑♓🌀❌_♓🔟♑♏💯♈♓🔴}📶♌_🔟⏫♓{❌⏫💯🔴♐🔴💯_♐✖}🔴♑📶♓✖♌🈲➗_♈♒🔟♊🔴❌_❌📶♏🚺_📶⛎♒{➗🌀⛎✖➿🚺♓_➿♏_🔴✖➿♊❌Ⓜ♐♉_🈲➗}➿♌♓💯➗✖♈🈲_Ⓜ🆔🔴{🔁♑♌💯♐♊♈_💯}🚺⏺🔁🚺➿💯➿➿♒_❌💯{➿➿♊🌀♈➿}⏫❌_⏫🌀♐🔁♊⛎♉♑📶🆔_♊♊{♓♑♑♓🔟♐♉🌀_♒♓}♌🔁♒📶📶♉❌🈲_♒🔴🔟♏⛎🆔♏{♑♑♏_♐}💯⏫🔴📶🔟Ⓜ♈💯♉_♊⛎{♑♑✖➿✖🌀⏫🚺_Ⓜ♌➿⏫♈⏫📶_♒✖_🔁♑♉🚺🔟🔟♒🔁📶🈲_♑🔴}🔁➿🆔➿💯💯🔟➗_⏺♉♏🔴🔟🔁🔴{➗🆔✖_♊🈲⏺🔁_🚺✖🆔❌🈲♓_❌💯{➿➿♊🌀♈➿}⏫❌_⏫🌀♐🔁♊⛎♉♑📶🆔_♊♊{♓♑♑♓🔟♐_♉🌀_♒♓}♌🔁♒📶📶♉❌🈲_♒🔴🔟♏⛎🆔♏{♑♑♏_🔟⏫♓{❌⏫💯⏫♓{❌⏫💯🔴♐🔴💯_♐✖}🔴♑📶♓✖♌🈲➗_♈♒🔟♊🔴❌_❌📶♏🚺_📶⛎♒{➗🌀⛎✖➿🚺♓_➿♏_🔴✖➿♊❌Ⓜ♐♉_🈲➗}➿♌♓💯➗✖♈🈲_Ⓜ🆔🔴{🔁♑♌💯♐♊♈_💯}🚺⏺🔁🚺➿💯➿➿♒_❌💯{➿➿♊🌀♈➿}⏫❌_⏫🌀♐🔁♊⛎♉♑📶🆔_♊♊{♓♑♑♓🔟♐♉🌀_♒♓}♌🔁♒📶📶♉❌🈲🔴♐🔴💯_♐✖}🔴♑📶♓✖♌🈲➗_♈♒🔟♊🔴❌❌📶♏🚺_📶⛎♒{➗_🌀⛎✖➿🚺♓_➿♏🔴✖➿♊❌Ⓜ♐♉_🈲➗}➿♌♓💯➗✖♈🈲_Ⓜ🆔_🔴{🔁♑♌💯♐♊♈_💯}🚺⏺🔁🚺➿💯_♊🈲⏺🔁_🚺✖🆔❌🈲♓_❌💯{➿➿♊🌀♈➿}⏫❌_⏫🌀♐🔁♊⛎♉♑📶🆔_♊♊{♓♑♑♓🔟♐_♉🌀_♒♓}♌🔁♒📶🌀🔴♌🔁♓🆔⏫⏫🔴⏫_Ⓜ🈲♊{⛎🈲⏺⏫🈲♈➗_🈲⏫🆔🆔}♊⛎➿⏫💯♓_🆔🈲{🔴🈲⏫♒♑♒Ⓜ}⛎_📶🔁♒♑♒♒✖🔟_💯♑_♈🆔♒🔴🔁🔁✖{♓📶♏_♈❌🌀❌🔁⏫♒♈}🔴♊_{♓💯♉💯⛎➿♑♓🌀❌📶♉❌🈲_♒🔴🔟♏⛎🆔♏{♑♑♏_🔟⏫♓{❌⏫💯🔴♐🔴💯_♐✖}🔴♑📶♓✖♌🈲➗_♈♒🔟♊🔴❌❌📶♏🚺➿➿♒_❌💯{➿➿♊🌀♈➿}⏫❌_⏫🌀♐🔁♊⛎♉♑📶🆔_Ⓜ♌➿⏫♈⏫📶🔴_♒✖_🔁♑♉🚺🔟🔟♒🔁📶🈲_♑🔴}🔁➿🆔➿💯💯🔟➗_⏺♉♏🔴_🔟🔁🔴{➗🆔✖_♊🈲⏺🔁🚺✖🆔❌🈲♓_♌♏➗{♉♈♌🚺❌⏫🌀_⛎♉♌✖♓📶🔟✖🚺♓_♌♓♒➗🔴♏♉♓⏺⏫_🌀♉}🈲✖♉🔟Ⓜ♒🌀♐_♏{♌♒➿♒✖♈♒💯♒_✖}🔴_♌🔟🆔❌🌀✖🔟❌_♌♓➿♊Ⓜ➿🔴💯♉Ⓜ_🌀♊Ⓜ➗{♌♒❌📶🚺💯_➗♌✖♉♐❌♊♌🚺♌_🌀🈲♑♌♓❌🆔➗♌}🔁_♈♉♊❌♓🔁♊♈❌💯{♊♉♒♈♓♈⏫♓_💯♏♉🌀_♐➿📶Ⓜ♏♑🌀}_♑💯{❌🚺♐♏⏺🔁♐🔁_♓🆔♌🔁📶📶❌🚺🔁✖_Ⓜ📶Ⓜ♏🌀♑Ⓜ♐🆔🔴_🚺⛎🆔🈲Ⓜ✖♒♌♑🆔_➗♈}🈲💯❌✖✖♐❌📶_♒❌♊{📶🆔♊➿⏺❌⛎_⏺❌⛎🚺⛎📶_🈲}🈲_♈🆔⏫{🔟♐⏫❌🔟🔴_🔴♈♉♒♌🔁✖🔴💯♒_📶}📶🆔📶📶➿🆔🆔💯🌀_⏺🚺♓♑⏺{🚺♓✖🔁♈_⛎♏♉Ⓜ}⏫⛎💯🚺_⏫🚺"""

for x, y in analysis.items():
    emoji_encoded = emoji_encoded.replace(x, y)

print(emoji_encoded)
```

#### flag
```
castorsCTF{sancoho_flag_qjzmlpg}
```
