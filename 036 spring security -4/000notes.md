# Notes 

|Feature|	Encoding	|Encryption|
|-------|---------------|--------------|
Purpose	|Data format transformation (for readability or compatibility)	|Data security (protecting confidentiality)
Goal	|Make data understandable to other systems (not secure)	|Make data unreadable to unauthorized parties (secure)
Key	|No key involved; uses standard algorithms (e.g., Base64, URL Encoding)	|Uses a key (secret or public/private) to encrypt and decrypt data
Reversibility	|Easily reversible by anyone who knows the encoding method	|Reversible only by someone with the correct key
Use Cases	|Data transmission, URLs, file formats (e.g., Base64 for images)	|Protecting sensitive data like passwords, credit cards, communication
Example	|Base64("hello") = aGVsbG8=	|AES("hello", secret_key) = garbled_text

### TL;DR:

Encoding is for data readability (not secure).

Encryption is for data confidentiality (secure).

### Encoding (Base64):
Original Message: hello

Encoded (Base64): aGVsbG8=

Anyone can decode aGVsbG8= back to hello using any Base64 decoder.

### Encryption (using a password/key):
Original Message: hello

Encrypted (AES with key = "abc123"): b94d27b9934d3e08a52e52d7da7dabfa (just an example of scrambled text)

Without the correct key, no one can understand or decrypt this back to hello.

Simple Difference:
Encoding = wrapping the message in a readable way.

Encryption = locking the message with a key.


![alt text](<036 spring security-4 basic auth_250716_001652_1.jpg>) 
![alt text](<036 spring security-4 basic auth_250716_001652_2.jpg>) 
![alt text](<036 spring security-4 basic auth_250716_001652_3.jpg>) 
![alt text](<036 spring security-4 basic auth_250716_001652_4.jpg>) 
![alt text](<036 spring security-4 basic auth_250716_001652_5.jpg>)
 ![alt text](<036 spring security-4 basic auth_250716_001652_6.jpg>) 
 ![alt text](<036 spring security-4 basic auth_250716_001652_7.jpg>)