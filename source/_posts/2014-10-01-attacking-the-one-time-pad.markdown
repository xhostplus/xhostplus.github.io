---
layout: post
title: "Attacking the One Time Pad"
date: 2014-10-01 15:29
comments: true
categories: 
- cryptography
- programming
---

**TL;DR;**
Using a One Time Pad is secure, ([perfectly secure actually](http://www.pro-technix.com/information/crypto/pages/vernam_base.html)), but you have to use the key _once and only once_.  

If the key is used again, also called a Two Time Pad, you can gather information on the messages sent.  This concept was used to crack [The Verona Project](http://en.wikipedia.org/wiki/Venona_project#Decryption), MS-CHAP in Windows NT, and the wireless protocol [WEP](http://en.wikipedia.org/wiki/Wired_Equivalent_Privacy#Flaws).

In this somewhat lengthy blogpost, I will demonstrate the insecurity of a Two Time Pad.  You will just need an understanding of XOR.

<!-- more -->

## Overview of the One Time Pad

The mechanics of the OTP are very straight forward.  You have a message _m_ of a certain length and a secret key _k_ of the same length.  To encrypt using the OTP: XOR _m_ and _k_ to get the resulting ciphertext _c_.  To decrypt: XOR _k_ and _c_ to get the message back.  Witness the power of XOR!

For example:
{% codeblock %}
ENCRYPT                  DECRYPT                  XOR Truth Table
m:  0 1 1 0 1 1 1        k:  1 0 1 1 0 1 0        a | b | a XOR b
k:  1 0 1 1 0 1 0        c:  1 1 0 1 1 0 1        ---------------
-----------------        -----------------        0 | 0 |   0
c:  1 1 0 1 1 0 1        m:  0 1 1 0 1 1 1        0 | 1 |   1
                                                  1 | 0 |   1
                                                  1 | 1 |   0
{% endcodeblock %}

Because nothing is perfectly perfect, there are some drawbacks to using the OTP in the real world:

1. It requires a key length as large as the message.
   If you wanted to encrypt the contents of a 1000 page book, you would also need another 1000 page book that represents the key.
+ The security of the OTP is only as secure as the key exchange.
   How does one send the key from Alice to Bob so that no one can intercept it?
+ Every key must be random, unpredictable and unique, which is a non-trivial software requirement.

## The Two Time Pad is Insecure

A Two Time Pad is when the key _k_ is used more than once.  If an eavesdropper gets a hold of these ciphertexts, information can be obtained.  Take these two ciphertext, _c1_ and _c2_.  Both encrypt different messages, but use the same key _k_.

_c1_ = _m1_ xor _k_  
_c2_ = _m2_ xor _k_

If the eavesdropper get these two ciphertext and XORs them together the key _k_ drops out (_a_ xor _a_ = 0), the result is:

_c1_ xor _c2_ = _m1_ xor _m2_

What is left if the difference between the two messages.  This can easily be shows with images instead of ones and zeroes.  Here is a screenshot of [ImageXOR](https://github.com/xhostplus/ImageXOR) program I wrote:
![ImageXOR_screenshot](/assets/ImageXOR_screenshot.png)

From this screenshot you can see both Message 1 and Message 2.  Both are encrypted using the same Key.  Both ciphertexts are shown, along with the difference between the two messages.  You can clearly see that the difference image contains information from both Message 1 and Message 2.  As an eavesdropper, we have obtained the information from both Messages.

Like I stated before, this is easily seen with images, the human brain can see patterns and images easily and we can pick out the images even though they are superimposed.  As a computer, this is a fairly difficult task.  How can we implement this on a computer to actually get the message text?

### Decrypting the Secret Message using the Two Time Pad

Luckily, the English language has enough redundancy that information can be obtained from a series of Two Time Pad ciphertexts.  More specifically, if we look at the [ASCII Table](http://www.asciitable.com/) we can see that if we XOR a lowercase letter with a SPACE, we get the uppercase version of that same letter.  The same holds true for an upper case letter.

{% codeblock %}
LETTER   HEX    BINARY
A:       0x41   0100 0001
Space:   0x20   0010 0000    XOR
-------------------------
a        0x61   0110 0001
{% endcodeblock %}

Using the above fact, we can take a few Two Time Pad ciphertexts and try to figure out what the secret messages are.  In the following example, this [secret message](/downloads/OTP/secret.txt) needs to be decrypted.  We are given [ten other ciphertexts](/downloads/OTP/ciphertexts.txt) that were encrypted using the same key _k_.

Compiling and running [this code](https://github.com/xhostplus/OneTimePad/tree/master/OTPAttack) against the ciphertexts will result in this output:

{% codeblock  Partially decoded ciphertexts %}
..E............MG.........._..I...A.......M..p........R........_..Y...........O..E.
..E.........H...G...........U...G..........E...D..........E...._..._...E....._.....
..E......Y...S..........H.......K_...N..............V.._.........E...O......N..N...
.........._...S....I....H....S..G..........E...D..........._..E..........T......C..
......C.....E.S..._....w..._..I.......E.........._......U.........U_...E...A....C..
..E....R........G.......H.........A.o....I...........E...S..........M..E.T..._.....
.H...E...._...S..E...._._.N...I.G.._....t....p....N.....U.._..E............A....C..
.H........_.........S......_...N..........M...........R......H......M....T.......E.
..._.........S........_..E..........o......E.......E....U.E............._..........
.....E..E..M...A...I....H......N..A...._...._...........U...T.......M..E..O.N......
{% endcodeblock %}

The code will compare each ciphertext message with every other one trying to determine where the spaces are.  Using the position in the ciphertext we believe to be a space, XOR that with the same position in the secret message.  Using this algorithm, we get the above output.  If we scan each column we can see some repeated characters, some are special characters, the . are non-printable characters and the - is believed to be a space.  From this information, we can try to reconstruct the secret message:

{% codeblock Partially decoded message %}
 he  ecre  message is  When using a One Time P d  never use the  ey mo e t an once  
{% endcodeblock %}

Remember that the capital letters will convert to lower case, and the lowercase characters will convert to uppercase. 
Filling in some of the missing letters we are almost done.
{% codeblock %}
The secret message is  When using a One Time Pad  never use the key more than once
{% endcodeblock %}

There are a couple of missing characters in this message, after _is_, after _Pad_ and at the end, after _once_.  We need to try to determine what the rest of the secret message is.  One way would be to use the secret message and decode the ciphertexts in the file.  We know the secret message and ciphertext, from this we can determine the key and decode the other ciphertexts.

_m_ xor _c_ = _k_

Compiling and running [this code](https://github.com/xhostplus/OneTimePad/tree/master/OTPDecode) using the same ciphertexts file, we can decode the secret key _k_ and the ciphertexts.  Using the partially decoded message we get this output:

{% codeblock Decoded Key (so far) %}
589C983EA55EF78A1F62970699D3C62E92639AC20A762D7844531FE208926C602E049ED462955E81F7F68890C94EBF5BBD591DBBC64A330A9E52ECA6E45807137CA5E7BE3B4540323BC8C305168260C7DB7E
{% endcodeblock %}

{% codeblock Partially decoded ciphertexts %}
In cryptography, encrcption is the process of enooding messages Or information in .
In symmetric-key schewes,[3] the encryption and hecryption keys Are the same. Thus.
In public-key encryptson schemes, the encryption,key is publisheD for anyone to us.
Encryption has long b.en used by militaries and kovernments to fAcilitate secret c.
TL;DR; Using a One Tiwe Pad is secure, (perfectlu secure actuallY), but you have t.
In this somewhat lengnhy blogpost, I will demonsxrate the insecuRity of a Two Time.
A Two Time Pad is whet the key k is used more thmn once. If an eAvesdropper gets a.
A publicly available jublic key encryption applioation called PrEtty Good Privacy .
For technical reasons6 an encryption scheme usua`ly uses a pseudO-random encryptio.
There is no way for tris person to know that the,message was altEred in it's encry.
{% endcodeblock %}

Looking at the decoded ciphertexts, there are some discrepancies which correspond to the missing characters in the partially decoded message.  In the first line of the decrypted ciphertexts, we can see that _encrcption_ should be _encryption_.  Here is how we can use the information so far to correct this and get the rest of the secret message:

1. Determine what the corrected key value should be
2. Adjust the current key with this corrected value
3. Determine the new decrypted value we are looking for

For the first discrepancy, we are looking at the 22nd character in the files:

{% codeblock Values for reference %}
Partially Decoded Key    = 589C983EA55EF78A1F62970699D3C62E92639AC20A|76|2D7844531F
Encoded Ciphertext       = 11F2B85DD72787FE7005E567E9BBBF02B206F4A178|15|5D0C2D3C71
Decoded Ciphertext       = I n   c r y p t o g r a p h y ,   e n c r |c |p t i o n
Partially Decoded Secret = T h e   s e c r e t   m e s s a g e   i s |  |  W h e n
{% endcodeblock %}

**_m_**   = The secret message value (The partially decoded message.  This will always be a SPACE in this example, since we are using a SPACE as a placeholder.)  
**_c_**   = The encoded ciphertext value (from the file)  
**_k_**   = The decrypted key value (after running the above program)  
**_k'_**  = The correct key  
**_k''_** = The adjusted key  
**_m'_**  = The correct secret message value  
**_m1_**  = The decoded ciphertext message value  
**_m1'_** = The corrected decoded ciphertext message value  

We know these values so let's set them:

**_m_**   = 0x20 (SPACE)  
**_c_**   = 0x15  
**_k_**   = 0x76  
**_m1_**  = 0x63 (letter c)  
**_m1'_** = 0x79 (letter y)  

1. Determine the correct key value:  
   We need to find the value of _k'_ that will give us the correct value, in this case the letter y:  
   **_c_ xor _k'_ = _m1'_**  
   Since XOR is commutative, we can re-write the equation to be:  
   **_c_ xor _m1'_ = _k'_**

+ Because of the malleability of XOR, we can adjust the old key value using the new key value.  This will produce a key value that will decrypt the SPACE character into the correct character in the secret message:  
   We have the correct key value, but we need to adjust it with the key value we already decrypted:  
   **_k'_ xor _k_ = _k''_**

+ Decrypt the SPACE value using the new _k``_ value:  
   Using this new _k''_ key, we can now compute the corrected letter for the secret message:  
   **_k''_ xor _m_ = _m'_**


Taking these equations and putting in the values we get the following:

0x15 xor _k'_ = 0x79 (y)  
0x15 xor 0x79 = _k'_ = 0x6C  
0x6C xor 0x76 = 0x1A  
0x1A xor 0x20(SPACE) = 0x3A (:)

If you are still following... we just deciphered the missing character in the secret message (:), and found the correct key value for that character (0x62).  Wash, Rinse, Repeat.  Here is the entire secret message:

{% codeblock %}
The secret message is: When using a stream cipher, never use the key more than once!
{% endcodeblock %}
