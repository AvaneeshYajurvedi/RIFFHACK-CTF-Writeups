
This was a question having RSA low exponent attack

I didn't have a complete image of how this works so i just first went to the target website

![Challenge Page](Images/Pasted%20image%2020260620000529.png)

we can see that 
- RSA modulus `n`
- Public exponent `e = 3`
- Ciphertext `c`

So i started researching about RSA and RSA low exponent attack methodology

RSA encryption normally works like this:


![RSA Encryption Formula](Images/Pasted%20image%2020260620000749.png)

where:

- `m` = plaintext
- `e` = public exponent
- `n` = modulus
- `c` = ciphertext

In this challenge 

e=3 which is a very small exponent

If the message is small enough, then:

![Low Exponent Observation](Images/Pasted%20image%2020260620000808.png)

Yeah i used AI for understanding this 

Okay so lets write a custom script for this 
It took some while to understand a lot of debugging 

```python
from gmpy2 import iroot
from Crypto.Util.number import long_to_bytes

c = 16061189851687597120996926301104657943371254567888802706085341545044609833111279875378049334585413773524402884958140237544824980651257894051011430259274146027654956734324320859907221246501733764510903950738295836450198242955006792630838088633440092969580163361765729135432881536101

m, exact = iroot(c, 3)

print("Exact cube root:", exact)
print(long_to_bytes(int(m)).decode())
```

Output:
```
Miles::bitctf{{sp1d3r_cub3d_br0adca57}}
```

Therefore the flag is :

```
bitctf{{sp1d3r_cub3d_br0adca57}}
```
