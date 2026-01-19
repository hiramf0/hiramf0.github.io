## UofTCTF 2026 - Baby (Obfuscated) Flag Checker

"All this obfuscation has left Han Shangyan seeing double. Even Grunt refuses to untangle this mess for him, so it's up to you to do the real _grunt_ work (hahaha get it???). 

Hint: This challenge can be solved without fully deobfuscating the script, but writing a deobfuscator might help you with the "ML Connoisseur" challenge...

Author: SteakEnthusiast"

This is a writeup of this rev challenge for the UofTCTF 2026.

The challenge contains an obfuscated Python script that checks for the flag with input().
Here's a small sample of the script:

```python
...
 def g0g0SQu1D(sQUId):
  gOg0SQUId = g0GOsquiD(G0g0sQu1D_116510(2874, 2812), G0g0sQu1D_116510(6168, 6986))
  while True:
      if gOg0SQUId == g0GOsquiD(G0g0sQu1D_116510(6391, 7896), G0g0sQu1D_116510(216, 1755)):
          while True:
              if GogOSQuId == g0GOsquiD(G0g0sQu1D_116510(3747, 900), G0g0sQu1D_116510(6757, 5359)):
                  break
              if GogOSQuId == g0GOsquiD(G0g0sQu1D_116510(2777, 6068), G0g0sQu1D_116510(5705, 2240)):
                  while True:
                      if gOG0Squ1D == g0GOsquiD(G0g0sQu1D_116510(7875, 53), G0g0sQu1D_116510(1882, 6860)):
                          GgS_178462 = sQUId * G0G0SQU1D(gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(73236485, 7617), G0g0sQu1D_116510(7156, 6421)), g0GOsquiD(G0g0sQu1D_116510(641, 9025), G0g0sQu1D_116510(9522, 132))), gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(5290, 7102), G0g0sQu1D_116510(4427, 1477)), g0GOsquiD(G0g0sQu1D_116510(7917, 1238), G0g0sQu1D_116510(9190, 9775)))) & G0G0SQU1D(gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(4294963250, 6335), G0g0sQu1D_116510(5588, 7195)), g0GOsquiD(G0g0sQu1D_116510(3975, 7813), G0g0sQu1D_116510(3915, 4094))), gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(414, 8731), G0g0sQu1D_116510(15775, 7006)), g0GOsquiD(G0g0sQu1D_116510(387, 6143), G0g0sQu1D_116510(6192, 1026))))
                          gOG0Squ1D = g0GOsquiD(G0g0sQu1D_116510(4297, 6513), G0g0sQu1D_116510(7495, 5945))
...
```

Going to the top of the script, we can find some interesting functions that can explain this script.
```python
def g0GOsquiD_37121(G0gosQuId, GOg0sQuiD):
    return ''.join((chr(G0g0squID ^ GOg0sQuiD) for G0g0squID in G0gosQuId))

def G0g0sQu1D_116510(gogoSQu1D, g0G0SQU1D):
    return gogoSQu1D ^ g0G0SQU1D

def G0goSQuId_531543(GOgosqUid, gOGosQu1d_408297, g0g0squ1D_281184, GOG0sQu1D):
    return (GOgosqUid ^ g0g0squ1D_281184) / (gOGosQu1d_408297 ^ GOG0sQu1D)

def gOg0sQuId_362335(G0goSQuId, G0gosqU1D):
    return g0GOsquiD_37121([], 124).join((chr(g0G0sQuId ^ G0gosqU1D) for g0G0sQuId in G0goSQuId))
```

Most of these functions appear to be obfuscated versions of regular XOR encryption functions:
```python
def XOR(a, b):
    return a ^ b
def decode(arr, key):
    return ''.join(chr(x ^ key) for x in arr)
```

Scrolling through the code, I found some interesting arrays in line 273-276 that could hold relevant information:
```python
GgS = (G0G0SQU1D(gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(2882, 3193), G0g0sQu1D [snip]
G0gosQu1D = [[G0G0SQU1D(gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(1861, 1988), G0 [snip]
sQU1D = [G0G0SQU1D(gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(1539, 3913), G0g0sQu [snip]
SqUId = [G0G0SQU1D(gOg0sQuId(g0GOsquiD(G0g0sQu1D_116510(11540, 3058), G0g0sQ [snip]
```

I used the Find & Replace function to change the function names in these variables to standardize everything. <br>
Many functions inside these declarations were the simple XOR functions mentioned above, so I converged all of the functions into the XO() and decode() functions inside my solve.py

### Writing solve.py
```python
def XO(a, b):
    return a ^ b

def decode(arr, key):
    return ''.join(chr(x ^ key) for x in arr)

GgS = (XO(7140, 7102) ^ XO(4949, 4969)) + XO(6864, 6855) & XO(6704, 6863)
G0gosQu1D = [[XO(2440, 2437), XO(865, 808), XO(5131, 5154), XO(6336, 6366 [snip]
sQU1D = [XO(2952, 2958), XO(6160, 6172), XO(27, 29), XO(8381, 8379), XO(8 [snip]
SqUId = [XO(7526, 7527), XO(3453, 3445), XO(1183, 1183), XO(8973, 8974),  [snip]
```

We can observe that `print(f"GgS = {GgS}") -> 125`, could this be a decryption key?
```python
for i, chunk in enumerate(G0gosQu1D):
    print(i, decode(chunk, GgS))
```
```bash
0 p4TcH_
1 uoftctf{d1d_
2 XD???}
3 d3BuG_
4 0n3_sh07_
5 4n_1LM_
6 r3v_0r_
7 th15_w17h_
8 y0u_m0nk3Y_
```

This is our flag! But, what's the order? Let's check the other 2 squid variables. <br>
When running `print(f"SqUId = {SqUId}")`, we get `SqUId = [1, 8, 0, 3, 6, 4, 7, 5, 2]`. <br>
If we align this array with the order of the snippets in G0gosQu1D, we get our flag: <br>

`uoftctf{d1d_y0u_m0nk3Y_p4TcH_d3BuG_r3v_0r_0n3_sh07_th15_w17h_4n_1LM_XD???}`

### solve.py
```python
def XO(a, b):
    return a ^ b

def decode(arr, key):
    return ''.join(chr(x ^ key) for x in arr)

GgS = (XO(7140, 7102) ^ XO(4949, 4969)) + XO(6864, 6855) & XO(6704, 6863)
G0gosQu1D = [[XO(2440, 2437), XO(865, 808), XO(5131, 5154), XO(6336, 6366), XO(6419, 6438), XO(981, 1015)], [XO(6379, 6371), XO(5105, 5091), XO(6312, 6323), XO(538, 531), XO(4355, 4381), XO(1758, 1751), XO(6775, 6764), XO(1315, 1317), XO(1178, 1155), XO(6273, 6349), XO(5190, 5215), XO(7864, 7834)], [XO(5318, 5347), XO(3003, 2946), XO(6517, 6455), XO(6735, 6669), XO(9498, 9560), XO(5645, 5645)], [XO(3085, 3092), XO(6136, 6070), XO(8776, 8823), XO(8954, 8946), XO(5909, 5935), XO(3709, 3679)], [XO(8449, 8524), XO(6551, 6532), XO(2189, 2243), XO(2478, 2444), XO(3063, 3065), XO(3006, 2987), XO(7847, 7914), XO(2347, 2401), XO(6738, 6768)], [XO(7420, 7349), XO(1178, 1161), XO(1710, 1676), XO(9477, 9545), XO(7807, 7758), XO(4031, 3983), XO(4028, 3998)], [XO(6522, 6517), XO(3607, 3673), XO(7602, 7609), XO(6132, 6102), XO(1857, 1804), XO(684, 675), XO(8158, 8188)], [XO(369, 376), XO(9127, 9138), XO(149, 217), XO(8100, 8172), XO(1900, 1870), XO(9196, 9190), XO(3939, 3887), XO(3892, 3966), XO(2946, 2967), XO(5065, 5099)], [XO(4106, 4110), XO(1949, 2000), XO(3307, 3299), XO(9441, 9411), XO(932, 948), XO(3705, 3636), XO(6821, 6838), XO(8170, 8188), XO(5053, 5107), XO(4532, 4496), XO(7979, 7945)]]
sQU1D = [XO(2952, 2958), XO(6160, 6172), XO(27, 29), XO(8381, 8379), XO(8910, 8903), XO(3542, 3537), XO(571, 572), XO(1552, 1562), XO(2508, 2503)]
SqUId = [XO(7526, 7527), XO(3453, 3445), XO(1183, 1183), XO(8973, 8974), XO(3816, 3822), XO(5125, 5121), XO(6336, 6343), XO(9185, 9188), XO(7188, 7190)]

print(f"GgS = {GgS}")
print(f"sQU1D = {sQU1D}")
print(f"SqUId = {SqUId}")

for i, chunk in enumerate(G0gosQu1D):
    print(i, decode(chunk, GgS))
```
