
# The Black Box of WinCrypto Opened

I've always struggled with cryptography implementations. There are many cryptography API tutorials openly available, but most of them stop before they are truly useful, and all of them tend to stick to one platform and/or language. The tough territory arrives when you are trying to bridge the gap between Windows and their official APIs, and everyone else.

Within the Windows environment, there are the WinCrypt API calls. These provide a fairly black box implementation of different cryptography functionality. There are a lot of hurdles to clear if you wish to use these APIs to fluidly interact with the Openssl command line tools, Python Crypto APIs, or other cryptography implementations.

## Public / Private Keys
Working with Public/Private keys would seem fairly simple. So long as you export your keys as a PKCS1 key with the CRYPT_OAEP flag, you are good. That CryptExportKey form will provide you with your keys in ASN.1 DER format. This is the simplest method to retrieve and work with Public/Private key crypto with Windows.

Exporting keys as SSL format is not as simple. The exported keys are in PRIVATEKEYBLOB or PUBICKEYBLOB format. It looks like there is a header here, with an RSA key right afterwards. Although these keys aren't usable in any of the non-Windows RSA implementations.

The format of PUBLIC and PRIVATE KEYBLOBs are similar.
```
| 0    -   3  | 4    -   7 | 8   -    ... |
|  BLOBTYPE   |    ALG ID  | Key Data     |
```
For BLOBTYPE:
* ```06 02 00 00 ``` represents PUBLICKEYBLOBs.
* ```07 02 00 00 ``` represents PRIVATEKEYBLOBS.

* ```00 A4 00 00``` represents the RSA algorithm.

Ideally you would be able to remove the 8 bytes making up the header and simply import the RSA Key. The issue is that each of the component parts within the RSA key structure are stored in Little Endian format, while openssl and other implementations expect the fields to be placed in Big Endian format.

The RSA Private Key follows the following format:
```
magic==RSA2
key length = DWORD [::-1]
public exponent = DWORD [::-1]
modulus = (size of keylength)[::-1]
prime1 = (key length / 2) [::-1]
prime2  = (key length / 2) [::-1]
exponent1 = (key length / 2) [::-1]
exponent2 = (key length / 2) [::-1]
coefficient = (key length / 2) [::-1]
privateExponent (size of keylength) [::-1]
```
The RSA Public Key follows the following format:
```
magic==RSA1
key length = DWORD [::-1]
public exponent = DWORD [::-1]
modulus = (size of keylength)[::-1]
```

## SESSION KEYS

Using the WinCrypt API for encrypted communications is more useful than file encryption, but is also more complex. Windows tries to protect the programmer from themselves by not providing any way to simply retreive the plaintext session key. CryptExportKey only allows you to retreive a SIMPLEBLOB format of the session key you've generated.

An example of the SIMPLEBLOB for an AES128 Session key can be seen in the image above.

The SIMPLEBLOB format is:
Bytes
```
| 0    -   3  | 4    -   7 | 8   -    11 | 12                       -                   ... |
|  BLOBTYPE   |    ALG ID  |             |                 Key Data                         |
| 01 02 00 00 | 0E 66 00 00| 00 A4 00 00 | NN  ...  NN  ...  NN   ...  NN   ...   NN ... NN |
|  SIMPLEBLOB |   AES-128  |  RSA_KEY_X  | Key data Little Endian encrypted with public key |
```

If you have the corresponding private key to the public key used to encrypt the Session Key, then retrieving the unencrypted Session Key is fairly simple in Python, but requires a few more steps in Win C++.


### Python Session Key Decoding
The following python will allow you to decode the session key:
```
from Crypto.PublicKey import RSA
import Crypto.Cipher.PKCS1_v1_5

priv = open('privatekey.der', 'r')
rsa_key = RSA.importKey(priv) # also works for PEM formatted keys.

c = Crypto.Cipher.PKCS1_v1_5.new(rsa_key)

aes_blob = open('simpleblob.bin', 'rb').read()
aes_enc = aes_blob[12:]
aes_key = c.decrypt(aes_enc[::-1], None)
```

To print the aes_key you can simply:
```
from binascii import hexlify
print(hexlify(aes_key))
```

Now that you have your AES session key, you need to instantiate the AES cipher correctly in order to use it. The Windows AES Session key uses CBC mode and an Initialization Vector of "0". This can be mimic'd in Python with the following call:
```
from Crypto.Cipher import AES
cipher = AES.new(key, AES.MODE_CBC, IV='\0'*16)
```


 ### Windows C+ Session Key Retrieval

Within a Windows program you can retrieve this same data via the following function. hProvider is the handle to your Crypto Service Provider. hKey is the handle to your AES Session Key. hExpKey is the handle to your imported, derived, or generated Public key for the Key exchange. cbData is the length of the key data

```
BYTE *CryptoHelper::SpcExportRawKeyData(HCRYPTPROV hProvider, HCRYPTKEY hKey, HCRYPTKEY hExpKey, DWORD *cbData) {
  BOOL      bResult = FALSE;
  BYTE      *pbData = 0, *pbKeyData;

	if (CryptExportKey(hKey, hExpKey, SIMPLEBLOB, 0, 0, cbData)){

		if ((pbData = (BYTE *)LocalAlloc(LMEM_FIXED, *cbData))){

			if (CryptExportKey(hKey, hExpKey, SIMPLEBLOB, 0, pbData, cbData)){
        /* Alternatively uncomment if you want to see the SIMPLEBLOB
				wprintf(L"AesKeyExport BLOB 2 = \"");
				for (int i=0; i < (*cbData); i++)
				{
					wprintf(L"%02x", pbData[i]);
				}
				wprintf(L"\"\n\n");
        */

				pbKeyData = pbData + sizeof(BLOBHEADER) + sizeof(ALG_ID);
				(*cbData) -= (sizeof(BLOBHEADER) + sizeof(ALG_ID));
				bResult = CryptDecrypt(hExpKey, 0, TRUE, 0, pbKeyData, cbData);
			}
			else{
				wprintf(L"Error Exporting Simple blob. err: %s\n", GetLastError());
			}
		}
		else{
			wprintf(L"Out of memory in export key\n");
		}
	}
	else{
		wprintf(L"Error in initializing key export. err: %s", GetLastError());

	}

	if (hExpKey) CryptDestroyKey(hExpKey);
	if (!bResult && pbData) LocalFree(pbData);
	else if (pbData) MoveMemory(pbData, pbKeyData, *cbData);
	return (bResult ? (BYTE *)LocalReAlloc(pbData, *cbData, 0) : 0);
}
```

You would instantiate this routine as such:
```
pbData = SpcExportRawKeyData(m_hCryptProv, hKey, hRSAKey, &cbKeySize);
wprintf(L"AesKeyExport Raw 2 = \"");
for (int i=0; i < cbKeySize; i++)
{
  wprintf(L"%02x", pbData[i]);
}
wprintf(L"\"\n\n");
```

A toy example of using the full Windows CryptoAPI for these items can be found here. It includes a lot of over explanation of what is occuring, but I found it useful for learning the details.

### References

If you are a Windows programmer, the simplest method to interact with these items via Python is with WinCrypto module found at: https://github.com/crappycrypto/wincrypto
Don't try to pip install this, as you will be left with a borked implementation as of January 2018. However the package installs fine if you git clone it and python setup.py install.

Much of my knowledge in this realm came from reading the code found within this implementation. After you understand how it works under the hood, you will likely only implement the necessary parts for your application.

When I was originally learning how to work with the Windows WinCrypt APIs I relied upon the examples on MSDN, and Hasherazade's AES_CRYPT.CPP implementation found here: hasherezade's Quick aes_crypt.cpp implementation:
https://gist.github.com/hasherezade/2860d94910c5c5fb776edadf57f0bef6

The code recipe 5.27 on page 246 of John Viega and Matt Messier's "Secure Programming Cookbook for C and C++" showed me the way towards decrypting the AES key from the SIMPLEBLOB within a Windows executable. I didn't like the goto's in their code, and rewrote it in if/else form. It's not quite as easy to read, but...goto...
