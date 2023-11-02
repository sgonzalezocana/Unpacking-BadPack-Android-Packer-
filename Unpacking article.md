# Unpacking-BadPack-Android-Packer-
Hello readers!

In today's article we bring you an analysis of a new packer that is currently being used in different banking malware families targeting Android devices such as Anatsa, Hydra or Cerberus. The fact that this packer has been observed in different families, makes us think that it is very likely that this is a new MaaS (malware as service). 

Malware packaging is commonly employed to hide the actual malicious payload of the analyzed sample, usually by encrypting it with any cryptographic algorithm. In this way, reverse engineering of the original sample is made difficult, since, as can be seen in the following image, the final file is split into two parts. On the one hand, it contains the actual encrypted malicious payload, and on the other hand, it contains the portion of code responsible for decryption and execution in memory of the malware.

IMAGE 1

This article will mainly deal with the process used to unpack BadPack Packer (so named due to the classification made by the antivirus engines in VirusTotal) through a Python script, which will provide us with the real malicious payload of the analyzed sample. This malicious payload is a file with extension .DEX, which stands for Dalvik Executable file and is so named because it is interpreted by the Dalvik virtual machine. Dalvik is part of Google's Android operating system and is used to run compatible applications.

## Initial analysis of samples

The analyzed sample was obtained from Virus Total and its SHA256 hash is as follows:

12f09fe5abe2cee128bcc551d39cddfd0bd2302fc9ded52c1489c92846ea8ce8

The analyzed sample has a class called StubApp, which is responsible for initiating the decryption and execution in memory of the malicious payload through the function renamed to "load_malicious_dex".

IMAGE 2

This function is in charge of looking for the "assets" folder inside the code and then searches it for the encrypted DEX file. Then, it decrypts it and loads it into memory through the "Invoke_dex_class_loader" function.

IMAGE 3

Initially we will try to locate the encrypted DEX file, which will be our object of study. To do so, the contents of the variable "dex_folder" and "dex_extension", which are encrypted in the following class, are searched:

IMAGE 4

The decryption function is a simple XOR algorithm:

IMAGE 5

The decrypted strings obtained are as follows:

IMAGE 6

Indeed, inside the 'qonig' folder you can find a file with extension 'wdj', which is the encrypted malicious payload.

IMAGE 7

Once the encrypted DEX file has been located, the decryption function used by the packer must be studied. In this case, the algorithm used is not any conventional symmetric cryptography algorithm such as RC4, AES or DES, but in this case the algorithm implemented is its own.

Initially, the algorithm performs a decompression through the ZLib library of the file.

IMAGE 8


Subsequently, the function "Init_Transformation" is called, which initially performs a series of operations on the variable "key", which is the decryption key of the DEX. As can be seen in the following image, it takes different values of the key and performs different OR operations and 16-bit shift to the left on them.

IMAGE 9

The equivalent Python code is as follows.

IMAGE 10

As you can see in the image, through ord we obtain the integer value that represents that unicode value. Finally, with the c_int32 function of the ctypes library you get the value represented as a 32-bit integer, since Java and Python interpret and represent signed numbers differently.

As seen in the following image, in Python this same operation returns the number 3197199208 (when in Java it returns -1097768088), which is the number BE916368 in hexadecimal.

IMAGE 11

Looking at a calculator, you can see that the representation of the hexadecimal number BE916368 is 3197199208 in decimal and -1097768088 in signed 2's complement. And this is the reason why the c_int32 function of the ctypes library must be used.

IMAGE 12

In the next part of the code it performs a transformation (Transformation1 function) on the array of integers obtained previously:

IMAGE 13

La transformación realizada es la siguiente:

IMAGE 14

El código Python equivalente sería el que se puede ver en la siguiente imagen.

IMAGE 15

Since Python does not have the unsigned right shift operator, it has to be implemented separately in a different function, which is called "unsigned_right_shift". On the other hand, the lsb32 (left shift) function has also been implemented, because as Python is able to handle large integers, the value obtained will be a smaller negative number in absolute value, which in this case is -26. However, as in Java numbers have a fixed size of 32 bits, the most significant bits will remain the same, and the remaining bits will be shifted to the right. Therefore, the result will be a signed integer, which in this case is 230, being this the value we are looking for. 

Subsequently, and as can be seen in image 13, the packer is responsible for reading the encrypted DEX file in portions of 8192 bytes to decrypt it little by little. Subsequently, it also uses a function which has been renamed to "Transformation2", to which it passes two arrays of integers previously calculated to continue performing operations on them. In this case, the operations performed are unsigned right shifts, OR operations, an addition and finally an XOR.

IMAGE 16

After all the operations performed, it saves in iArr2[0] and iArr[1] a pair of values, which will be used in the final phase of decryption:

IMAGE 17

To finish decrypting the file, it makes an XOR of the result obtained between the operation "opp1" and the byte of the array on which it is iterating. Finally, in "opp3" what is being done is to pass the value obtained to byte.

IMAGE 18

Once the array of bytes that make up the complete file has been obtained, a Zlib inflate is performed again on it, obtaining the decrypted DEX file, which is the actual payload of the malware.

As can be seen in the following image, the bytes obtained have the header "dex", which indicates that the decryption has been performed correctly.

IMAGE 19

Therefore, this is a case in which the decryption algorithm has been implemented by the malware developers themselves, making it difficult to decrypt the malware.

And this is the end of today's post. I hope it has been useful and that you liked it, see you next time!

## References

https://medium.com/@dbragetti/unpacking-malware-685de7093e5
https://filext.com/file-extension/DEX

