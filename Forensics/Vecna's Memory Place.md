
This was a tricky question too took me some time to figure out 

the challenge page is shows something like this 

<img src="Images/Pasted%20image%2020260620005159.png" width="700">


lets retrieve the incident snapshot 
a file named memory_snapshot.bin gets downloaded lets check it

```
file memory_snapshot.bin  
memory_snapshot.bin: ASCII text
```

so its a ASCII text file okay that works 

lets view it 

```
cat memory_snapshot.bin  
=== HAWKINS_LAB_VOLATILE_CAPTURE ===  
proc=svchost.exe pid=1948 parent=services.exe  
driver=.tmp\mindflayer\resident.sys state=orphaned  
radio_tag=UNJX  
tv_tag=VAF  
clockface=13  
DEC0Y_PIPE=Vecna\Clock_One\ringbuffer  
noise_0:nInxPHet2cbSHn2yPVL0NCqS  
memfrag[4]=UklEPTR8UE9TPTR8U0laRT01M3xCTE9CPTcxNjA3ZDc5N2I2YjdkNzIzMjJmN2U3ZDMxNzA3MTZlMmE2YjYyNzEzZDJmM2IyNDJhMjUw  
cache_shadow[0]=UklEPTh8UE9TPTR8U0laRT0xMnxCTE9CPWRlYWRjYWZlYmFiZQ==  
noise_1:U054PXwLexPCKEBhiR7xLzJy  
memfrag[1]=UklEPTF8UE9TPTN8U0laRT01M3xCTE9CPTY2OTczNmM2NDJlNzgzNDc5MmIyZjY3MmQ3OTZmMmQ3MTdjNjQyYzc3NjY3ZDc5N2E2Njdk  
cache_shadow[1]=UklEPTJ8UE9TPTl8U0laRT0xMHxCTE9CPTQxNDE0MTQxNDE=  
noise_2:oiYVPQ5rAO3V7GS2Vqy9h1Wl  
memfrag[5]=UklEPTV8UE9TPTJ8U0laRT01M3xCTE9CPTNhM2EzMjNhMjIzODNlM2IzYTBjMjYyODMwMjMzZDNkM2IyMTI3MjM2OTY1NmMyMDIwMjA2  
noise_3:cc8fSspC/O0rb1rOzSkfa56K  
memfrag[0]=UklEPTB8UE9TPTF8U0laRT01M3xCTE9CPWIyMTE3MmQzODJhMmQyYjIxNjYyNTNiMjc2YjYyNzEyYjIwM2EzYjI4MjczNDI2NjM2ZDY5  
noise_4:G/BuWOA3uS9HoKXY12jIk5US  
memfrag[2]=UklEPTJ8UE9TPTV8U0laRT01M3xCTE9CPWMzYzJlM2MyZTI3NmM2OTZhMTIxNDA0MDYxZTAwNjUwMDFmMDQxMDYzNjI3MTc5NjI2OTM0  
noise_5:bQbZGWckADMyJzMrxwhQ9NNp  
memfrag[3]=UklEPTN8UE9TPTB8U0laRT01M3xCTE9CPTMzNjMzNjM5M2QyNzM1MjkyMjIzMTQyNzJmM2UyZDYzNmQ2OTI0MjczZDJjMjczYjJhMzAy  
diag=heap snapshot complete  
analyst_note=fragment metadata survived even when the slab order did not  
=== END_CAPTURE ===
```

The first thing I do with any unknown forensic file is extract readable strings.

```
strings memory_snapshot.bin
```

Among the output were some interesting entries:

```
radio_tag=UNJX
tv_tag=VAF
```

and several entries that looked like:

```
memfrag[0]=...
memfrag[1]=...
memfrag[2]=...
memfrag[3]=...
memfrag[4]=...
memfrag[5]=...
```

	The challenge title is:

```
Vecna's Memory Palace
```

The description mentions:

```
Hawkins Lab
```

which is a Stranger Things reference.

The strings:

```
UNJX
VAF
```

look random.

A common CTF trick is ROT13.

ROT13 shifts letters by 13 positions.

Examples:

```
A <-> N
B <-> O
C <-> P
```

Applying ROT13:

```
UNJX -> HAWK
VAF  -> INS
```

Combining both:

```
HAWK + INS = HAWKINS
```

Now we have:

```
HAWKINS
```

This is probably important because:

- Hawkins is mentioned in the challenge story.
- The decoded word makes perfect sense.
- CTF authors often hide encryption keys this way.

At this point I suspected:

```
HAWKINS
```

is the decryption key.

The file contains several fragments:

```
memfrag[0]
memfrag[1]
memfrag[2]
memfrag[3]
memfrag[4]
memfrag[5]
```

Each fragment is Base64 encoded.

Decode them

Each decoded fragment contains:

```
RID
POS
SIZE
BLOB
```

Notice that:

```
RID ≠ POS
```

This means they are intentionally shuffled 

The challenge wants us to rebuild the original data.

So we sort by:

```
POS
```

Example:

```
POS=0
POS=1
POS=2
POS=3
POS=4
POS=5
```

After sorting, concatenate all BLOB values together.At this point:

- We have a mysterious encrypted blob.
- We have a suspicious word:

```
HAWKINS
```

The most common possibility is XOR encryption.

```
Repeating-key XOR
```

where the key repeats forever.

Key:

```
HAWKINS
```

ASCII values:

```
H = 48
A = 41
W = 57
K = 4B
I = 49
N = 4E
S = 53
```


After XOR decryption, the data becomes readable.

The output contains a JSON object:

```
{
  "artifact_name":"mindflayer_loader.dll",
  "campaign":"starcourt_nightshift",
  "sha1":"7f9c2ba4e88f827d616045507605853ed73b809a",
  "unlock_token":"SCOOPS-AHOY-1985"
}
```

Fill the form with:

## artifact_name

```
mindflayer_loader.dll
```

## sha1

```
7f9c2ba4e88f827d616045507605853ed73b809a
```

## unlock_token

```
SCOOPS-AHOY-1985
```

Submit the form.

Challenge solved.

```
{ "flag": "bitctf{{v3cn45_m3m0ry_p4l4c3_br34ch}}", "status": "validated" }
```

Flag

```
bitctf{{v3cn45_m3m0ry_p4l4c3_br34ch}}
```
