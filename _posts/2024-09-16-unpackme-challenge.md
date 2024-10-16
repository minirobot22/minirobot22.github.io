---
title: [Uncovering New Techniques through solving "UnpackMe" Challenge]
date: 2024-09-16 00:00:00 +0800
categories: [Digital Forensics]
tags: [dfir,ir,malware analysis]     # TAG names should always be lowercase
---


As a malware analyst, we received a sample 
Often times as a malware analyst, I receive vague file samples and get asked to analyze them. What I mean by vague is that they either unknown to most Public threat intelligence platforms or they do exist with low/high detection rate but with generic detection names. Those are mostly hueristix detections which..

In all cases, I usually perform the analysis with the following goals in mind:
* Determine if the sample is actually doing something harmful, that is true or false positive.
* The second most important thing to be concerned about is the malware capabilities & the risk associated with it

In our current scenario, we are mostly going to uncover & highlight some of the main capabilities of the sample and answering the challenge questions as we progress.
Along the way, we will learn various techniques and work arounds to tackle some of the challenges encountered that hindered our dynamic analysis of the malware.

First of all, I`d like to examine my sample using basic static analysis, throwing the sample on PEstudio, DIE for quick triage, will not spend too much time in this phase as the sample seems to be packed, no visible strings, entropy seems to be high which indicates some degree of packing.

![Entropy](/assets/images/unpackme/Entropy.png)
_Entropy_

We saw <code class="highlighter-rouge">VirtualAlloc</code> as one of the API calls being imported from the imports address table of the sample, so,
Load the sample in x32dbg, and let`s simply start by adding a breakpoint on VirtualAlloc


![VirutallAlloc](/assets/images/unpackme/[2].adding_breakpoint.png)
_ViruallAlloc_


After we run the program, we hit our breakpoint

![VirutallAlloc](/assets/images/unpackme/3.VirtualAlloc.png)
_ViruallAlloc_

Continue until we return back from the system dlls (kernal32,etc), and observe the address returned in EAX

![EAX Address](/assets/images/unpackme/4.Follow.png)
_EAX Address_

Allocated space by VirtualAlloc is currently empty, stepping over a few instructions, particularly after the next call (call 763E52 in my case), the memory address started to populate, this memory segment will contains the unpacked binary that the malware execution will be transferred to.

![Empty Allocated Space](/assets/images/unpackme/3.EXTRA-Empty.png)
_Empty Allocated Space_

![Function 763E52](/assets/images/unpackme/5.Dump-deobfuscation-call.png)
_Function 763E52_

Scrolling down a bit, we can see the start of an executable file (MZ header).

![MZ Header](/assets/images/unpackme/6.MZ-Header-After-Scrolling.png)
_MZ Header_

we can follow on memory map, then right-click on the specified memory region and dump it to a file

![dump memory region](/assets/images/unpackme/7.dump-it.png)
_dump memory region_

Open the dumped file in any hex editor and get rid of the extra bytes before the MZ header, then save the file

![Removing Extra Bytes](/assets/images/unpackme/8.remove-extra-HxD.png)
_Removing Extra Bytes_


Now, we should have the clean unpacked sample, and to make sure, we can inspect its imports and strings again. In fact, from the strings alone, we have a pretty good idea about what the malware sample might be doing (information stealer) as we see below

![Strings](/assets/images/unpackme/[9].strings-1.png)
![Strings](/assets/images/unpackme/[9].strings-2.png)
_strings_

Among other things, strings also reveal the internal path of the project build 

![Strings](/assets/images/unpackme/[9].strings-3.png)

Furthermore, if we search for some of the unique strings it would give away the sample varient which is one varient of RacoonStealer.

Now, let`s start examining the malware by Loading the unpacked sample into IDA. The malware begins by creating a mutex (function sub_DD2DF7) to prevent the malware from being executed twice on the same host.

![Main Function](/assets/images/unpackme/10-MAIN-Function.png)
_Main Function_

We can see the API OpenMutexA inside the function, with ESI register holding the mutex name.

![CreateMutex](/assets/images/unpackme/11.RenamingFunction.png)
_CreateMutex_


If we debug the sample using IDA, and we examine the address at ESI, we can obtain the Mutex name as below

![IDA Debugging to Find Mutex Name](/assets/images/unpackme/12.MUTEX-ESI-string.png)
_IDA Debugging to Find Mutex Name_

![Mutex Name](/assets/images/unpackme/13.Mutex-STRING.png)
_Mutex Name_


Next, the malware calls (435AE5) which I renamed CheckPrivs below to check if the process is running with LOCAL SYSTEM privileges, if so, it will proceed by finding explorer.exe process, duplicate its token and execute with its privileges.

![checking Priviliges](/assets/images/unpackme/13.2.check-privs-then-compare.png)
_checking Priviliges_

Here is the portion within the next function (435B8A), renamed Findingexplorerprocess where it is trying to find explorer.exe

![Comparison Loop](/assets/images/unpackme/14.0.loop-comparison-with-explorer-from-ID.png)
_Comparison Loop_

Comparison looking for explorer.exe is shown below from X64dbg

![X64dbg looking for explorer process](/assets/images/unpackme/14.comparison-with-explorer-from-x64dbg-to-show-actual-values.png)
_X64dbg looking for explorer process_

Next up, the execution will continue to find the local machine language by calling GetUserDefaultLCID and GetLocaleInfoA.

![Checking Language](/assets/images/unpackme/16.Checking-language-portions-repeated-section-of-XOR.png)
_Checking Language_

Then it dynamically uses XOR in a bunch of loops to check a set of languages (Russia, Kazakhstan, Uzbekistan, etc) if they match, the malware will stop execution and exit.
The malware uses repetitive XOR blocks to decrypts each segment during execution. At the block location (loc_429AE3), before jumping into a different block of code that appears to have some base64 strings, it reveals a special string after finishing the XOR loop.

![Last XOR spits out the key](/assets/images/unpackme/17.Key_from_IDA.png)
_Last XOR spits out the key_

The key is revealed in the memory segment below

![RC4 Key](/assets/images/unpackme/15.111.last-loop-of-XOR-spits-out-the-KEY.png)
_RC4 Key_

This will turn out to be the RC4 key it uses for encrypting all C2 communications.
In the next block, we see three base64 strings

![Three Base64 strings](/assets/images/unpackme/17.2.00some-base64-being-pushed.png)
_Three Base64 strings_

Stepping over in x64dbg until we hit function (DB47F6), after the function call, we immediately notice a C2 domain pops up

tttttt.me/ch0koalpengol

![staging domain appearance](/assets/images/unpackme/17.3.Hit-by-the-fun-7F6-after-that-we-saw-the-resolution-of-a-domain-in-EAX.png)
_staging domain appearance_

Let`s step back for a moment and look back before the function gets called, the function had two parameters that were pushed into the stack, one is the base64 decoded value of the first base64 string shown previously, the other is the RC4 Key we discovered.

![RC4 Decryption Function](/assets/images/unpackme/17.4.Take-a-look-at-the-function-args-it-takes-the-key-and-the-decoded-base64.png)
_RC4 Decryption Function_

Therefore, we know that this function is using the decoded string along with the RC4 key to decrypt its first C2 domain.
If we look back even further, we can get the function that performs the base64 decoding, let`s rename & label those two important functions.
 
![Renaming Functions](/assets/images/unpackme/18.1.base64_and_RC4_decrypt.png)
![Renaming Functions](/assets/images/unpackme/18.0.Base64_fn.png)
_Renaming Functions_

Digging deeper into the RC4 Decryption function, we notice 4 subroutines, from there we can identify the RC4 algorithm by checking both functions (41468E) and (414712)

![Inside RC4 Decryption Function](/assets/images/unpackme/18.3stepping-into-this-func-in-IDA-we-see-4-subroutines.png)
_Inside RC4 Decryption Function_

below screenshots clearly shows the key Scheduling Algorithm (KSA) generation  of the RC4 (the 256 loop is a quick giveaway), then the key stream is being XORed with each byte of the payload in the other function.

![RC4 Algorithm](/assets/images/unpackme/RC4-algorithm.png)
_RC4 Algorithm_


Going further after RC4 decryption of the domain, we should expect to see some network communication to that C2 domain, this takes place at function (430F27) as we see below 

![HTTP REQ Function](/assets/images/unpackme/21.HTTP_REQ_Func.png)
_HTTP REQ Function_

Next function appears to be checking for certain strings in the response body of the HTTP request. We clearly see some html tags "description and dir=auto>

![HTTP Response Comparison](/assets/images/unpackme/22.0.The-function-call-for-resposnce-comparison.png)
_HTTP Response Comparison_

![HTTP Response Comparison](/assets/images/unpackme/22.1.The-function-call-for-resposnce-comparison-IDA.png)
_HTTP Response Comparison_

In order to make sense of it, we checked the C2 link that the malware was trying to request(tttttt.me/ch0koalpengol) , then looking for the html strings needed, nothing was found. It appeared that the C2 was not active at the time of our analysis anymore.

![Telegram Page](/assets/images/unpackme/the-telegram-page.png)
_Telegram Page_

![Telegram HTML Page](/assets/images/unpackme/the-telegram-code.png)
_Telegram HTML Page_

As a result of that, the malware was stuck at the next block of code, Which is a loop that sleeps for 5 sec. , then continues to request the domain until the string match is found.

![HTTP REQ Loop](/assets/images/unpackme/22.2.The_HTTP_loop_where_we_first_stucked.png)
_HTTP REQ Loop_

Apparently, this hindered our dynamic analysis as the malware was not able to retrieve this string to continue its operation.
So, we have to come up with a different approach to emulate this on our local network to further continue debugging and discover the rest of the stealer operation.


MALWARE C2 EMULATION
-----------------------------------------

So far, we have understood how the malware uses RC4 encryption to decrypt its C2 communications, and we have also seen that the base64 string (qSVdAbi/K2pP9eTPjNld5MgaAL+bQsyox4MDv0iVTuA=) was actually the telegram C2 domain after decoding and decrypting it. In order to continue debugging, we have to alter the code a bit and replace this base64 string with our own, which would resemble our own C2 on a local network.
To do that, let`s setup an http server on our localhost, prepare similar C2 page by copying the telegram html code to our new page, and add the missing HTML strings that the malware would expect to retrieve.

I have added a simple string for testing, Mine would be saved and hosted at localhost/telegram.html


![Local Modified Copy of Telegram Page](/assets/images/unpackme/23.0.emulated_the_telegram_page_on_our_local_server_and_added_the_expected_response_string.png)
_Local Modified Copy of Telegram Page_


knowing the RC4 key, we can assemble our base64 with the help of CyberChef as below

![CyberChef Base64](/assets/images/unpackme/23-0000modifying_the_telegram_c2_for_emulation.png)
_CyberChef Base64_

Now, let's edit our malware file with a hex editor and search for the base64 string (qSVdAbi/K2pP9eTPjNld5MgaAL+bQsyox4MDv0iVTuA=), our goal is to replace that portion with our own base64 so that after malware decodes it, we get our own telegram.html page.

![Malware with HexEditor](/assets/images/unpackme/23.00-HexEditor.png)
_Malware with HexEditor_


After highlighting the reqiured string, Write-click, then choose paste write and save the file.

Let`s test it by going to the RC4 Decryption instruction (*7F6) and continue debugging, we should see our telegram page being resolved. 

![Local Telegram Page](/assets/images/unpackme/pic.png)
_Local Telegram Page_

Now, if we continue debugging from where we left off, we should jump at the block after the HTTP request, and our embedded string should appear as shown below

![Modified String](/assets/images/unpackme/23.1.laterom-string.png)
_Modified String_

Next, if we step over a few instruction, we would start to see the string being filtered.

![Filtered String](/assets/images/unpackme/23.1.laterom-string.png)
_Filtered String_


If we look closer after the function that extracts the string, we would see two calls to the same function (renamed filter_out here). In the first call, the value 5 is being pushed into the stack before the function call, and value 6 in the second function call, which corresponds to filtering out the first 5 characters from the strings, and the last 6 characters respectively.

![Filtered String Function](/assets/images/unpackme/23.3.After_Extracting_Value_it_gets_filtered_Twice_beginning_and_end.png)
_Filtered String Function_

Here is the code inside the function

![Filtered String Function](/assets/images/unpackme/23.4.Inside_the_first_filter_out_functions.png)
_Filtered String Function_

Moving on, after our string was filtered, we notice another calls to both base64 decoding & RC4 decryption funcions with a reference to one of the hardcoded base64 strings shown earlier.

![2nd RC4 Decryptian to get the main C2](/assets/images/unpackme/23.5.2nd-RC4-Decryption-of-C2.png)
_2nd RC4 Decryptian to get the main C2_


We know that RaccoonStealer uses the middle stage we saw earlier (telegram channel) to retrieve its main C2 domain from the string that was missing in our case.

In the screenshot above, the RC4 Decryptian function (RC4 Decrypt) is used to decrypt our retrieved and filtered string using the referenced hardcoded base64 string as the RC4 key and the resultant output should be the malware main C2.


obviously,  our test string currenly does not make sense to the malware opertion. So, we can simulate the operation by reversing the process, similar to what we have done with the telegram domain. Given the new RC4 key, we can asssume a C2 domain hosted in our local server (localhost/dump.php), encrypt it with the RC4 Key (6af7fae138b9752d1d76736dcb534c9d) and produce a base64 sting that can be replaced by our test string in our telegram HTML page.

![producing a main C2 in base64](/assets/images/unpackme/This_time_we_will_use_the_6af_key_instead_for_the_2nd_C2_comm.png)
_2nd RC4 Decryptian to get the main C2_

Now, this generated base64 will act as the main C2, that will be replaced with our old string, remember we have to pad our string with 5 characters at the beginning, 6 at the end, so that when it gets fitlered, we get the correct domain.

![Main C2 Base64 String](/assets/images/unpackme/24.Modifed_the_string_for_the_2nd_c2_to_dump_the_request_to_our_local_server.png)
_Main C2 Base64 String_


let`s puase for a moment and understand how this stealer works and how it usually communicates to its C2.


RacconStealer C2 Operations
-----------------------------
We can understand the C2 operations from various samples that exist on public sandboxes like any.run.

![C2 Operation](/assets/images/unpackme/24.8-C2-Operations.png)
_C2 Operation_

RacconStealer operations usually works by following the below sequence:

1. POST Request to its main C2 server (localhost/dump.php in our case) with identification paramters in the body (botID, configID, etc).

2. The server replies with important malware configuration including a URL for the malware to download additional DLLs required for its stealing operations.


![Malware Config with the URL](/assets/images/unpackme/24.1.Server-Reply-with-config.png)
_Malware Config with the URL_

3. The malware Download/Request the required DLLs (legitimate DLLs) and continue by collecting host information based on its config, finally, exfiltrate all data to the C2 server.

We could use a proxy to capture the malware request to the server and observe the payload, in our case, we prepared a PHP (duip.php) page that will dump the request to a file on disk.

A very handy php script that can perfrom this can be found here

https://gist.github.com/magnetikonline/650e30e485c0f91f2f40

Now, if we return back to our x32dbg and continue after the RC4 Decrypt function of the main C2 (duip.php), few instructions later we hit the first POST request to the server.

![POST Request](/assets/images/unpackme/24.999-2nd-post-duip.png)
_POST Request_

The dumped request is shown below, The payload contians the Bot-ID and Machine ID ecrypted with the same RC4 key.

![First Request dumped](/assets/images/unpackme/23.6-verify-thepost-reuqest-being-dumped.png)
_First Request dumped_


looking closer at the script (duip.php), it returns the string "Done" which is not expected by the malware, this will cuase the malware to stop and exit later on when it tries to check the server response. 

![Return Value](/assets/images/unpackme/23.return-value.png)
_Return Value_


After the below section which checks the return values, malware exists.

![Checking Configuration](/assets/images/unpackme/23.7-check-config.png)
_Checking Configuration_

To fix this, we need to edit the response wihthin the script, and provide the malware with an expected payload response (we got a fake response from the same sandbox sample shown earlier).


[pic of the response from sanbox]


![Replacing string with our payload](/assets/images/unpackme/23.9.duip-script-edited.png)
_Replacing string with our payload_


As mentioned, after decoding this payload with the RC4 Key, it will contain several different DLLs to Download to the infected host. Those DLLs would then be placed in the AppData\LocalLow location under the user profile.

![Malware Config with the URL](/assets/images/unpackme/24.1.Server-Reply-with-config.png)
_Malware Config with the URL_

![Payload decoded from X32dbg](/assets/images/unpackme/24.999999-decoded.png)
_Payload decoded from X32dbg_


Since we don`t have the correct URL to download the required DLLs and we are emulating the stealer on our own network
we obtained the files from different available RaccoonStealer sample found on any.run sandbox. 

![DLLs Zipped](/assets/images/unpackme/24.9-Download-DLLs.png)
_DLLs Zipped_

Then, we placed the files in the location where the malware expects to find them, that is under locallow directory.

![Files dropped in LocalLow folder](/assets/images/unpackme/23.8-drops-to-locallow.png)
_Files dropped in LocalLow folder_

Those are legitmate third-party DLLs required by the malware to gather information about the infected machine, some of the DLLs includes softokn3.dll
, sqlite3.dll, nss3.dll, nssdbm3.dll and others.


DLLs were placed inside LocalLow\rer as below

[DLLs pic]


At this point, the malware would perform the bulk of its stealing functionality, including stealing passwords, credit card information, browser cookies, history, etc.

As an example, sqlite3.dll is utilized to perfrom SQL commands to steal information from browsers of the infected machine.

![sqlite3.dll](/assets/images/unpackme/5000.2sqlite3.png)
_sqlite3.dll_


continue debugging with X32dbg, the malware would successfully dump a file named machineonfo.txt  under locallow direcroty, this file would contain machnine information including the RaccoonStealer version number.

![Machineinfo.txt](/assets/images/unpackme/50000000.3_machineinfo.txt.png)
_Machineinfo.txt_


Turning our attention to registry key function calls, if we search for openRegKey we get multiple calls, focusing on the function highlited below


![Looking for RegOpenKey](/assets/images/unpackme/30-Function-Call-to-openregkey.png)
_Looking for RegOpenKey_

navigating to that function call, we can immeditaly observe a common registry key used to check installed software running on the host

![Installed Software Key](/assets/images/unpackme/30.1.the-registry-key-uninstall.png)
_Installed Software Key_

Malware commonly use the API call (GdipSaveImageToFile) to save an image to a file, so, examining this section of code, we can tell that it is used to peform scree capture from the infected host

![GdipSaveImageToFile](/assets/images/unpackme/39-The-Function-respnsible-with-capturing-screenshots.png)
_GdipSaveImageToFile_

Checking this API Call cross references, we can obtain the main function address

![screencapture address](/assets/images/unpackme/9000-screencapture.png)
_screencapture address_

Finally, The malware removes all its traces by executing the following command, which can be found with many instances of the stealer online

![Malware Deletion](/assets/images/unpackme/6000-COMMAND-TO-DELETE-ITSELF.png)
_Malware Deletion_