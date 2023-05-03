Download Link: https://assignmentchef.com/product/solved-cs3339-lab-7-buffer-overflows
<br>
<h1>Introduction</h1>

<strong> </strong>In this lab, you will learn how buffer overflows and other memory vulnerabilities are used to takeover vulnerable programs. The goal is to investigate a program I provide and then figure out how to use it to gain shell access to systems.




In 1996 Aleph One wrote the canonical paper on smashing the stack. You should read this as it gives a detailed description of how stack smashing works. Today, many compilers and operating systems have implemented security features, which stop the attacks described in the paper. However, it still provides very relevant background for newer attacks and (specifically) this lab assignment.




Aleph One: Smashing the Stack for Fun and Profit:

<u>https://web1.cs.wright.edu/people/faculty/tkprasad/courses/cs781/alephOne.html</u>

Another (long) description of Buffer Overflows is here:

<u>http://www.enderunix.org/docs/en/bof-eng.txt</u>




<h2>Software Requirements</h2>




<ul>

 <li>The VirtualBox Software</li>

 <li>The Kali Linux, Penetration Testing Distribution</li>

 <li>GDB: The GNU Project Debugger</li>

 <li>GCC, the GNU Compiler Collection</li>

 <li>C source file including BOF.c, createBadfile.c, and testShellCode.c</li>

</ul>

<h1>Starting the Virtual Machine</h1>

<strong> </strong>

You can use the Kali VM from Lab 1










Login the Kali Linux with username root, and password CS3339

Download the following 3 files on your VM: BOF.c createBadfile.c and testShellCode.c  <u>https://s2.smu.edu/~rtumac/cs3339/spring2020/Lab7/</u>







<h1>Setting up the Environment</h1>

<strong> </strong>

There are many protections in current compilers and operating systems to stop stack attacks like the one we want to do. We have to disable some security options to allow the exploit to work.

<h2>Disable Address Space Layout Randomization</h2>

Address Space Layout Randomization (ASLR) is a security features used in most Operating system today. ASLR randomly arranges the address spaces of processes, including stack, heap, and libraries. It provides a mechanism for making the exploitation hard to success. You can configure ASLR in Linux using the

/proc/sys/kernel/randomize_va_space interface. The following values are supported:

<ul>

 <li>– No randomization</li>

 <li>– Conservative randomization</li>

 <li>– Full randomization Disable ASLR, run:</li>

</ul>

<em>$ echo 0 &gt; /proc/sys/kernel/randomize_va_space </em>Enable ASLR, run:

<h3>$ echo 2 &gt; /proc/sys/kernel/randomize_va_space</h3>

Note that you will need root privilege to configure the interface. Using vi to modify the

interface      may    have   errors.          The    screenshot    below shows         the     value of /proc/sys/kernel/randomize_va_space

However, this configuration will not survive after a reboot. You will have to configure this in sysctl. Add a file /etc/sysctl.d/01-disable-aslr.conf containing:

<em>kernel.randomize_va_space=0 </em>

This will permanently disable ASLR. To achieve the above, you can run the following:

<h3>$ touch /etc/sysctl.d/01-disable-aslr.conf $ echo kernel.randomize_va_space=0 &gt; /etc/sysctl.d/01-disable-aslr.conf</h3>

<em> </em>







The screenshot below shows you the ASLR configuration. You can open a terminal and try it out.













<h2>Install GDB and GCC Multilib</h2>

<strong> </strong>

<strong>          </strong><em>$ sudo apt update </em>

<em> </em>

<em>          $ sudo apt install gdb gcc-multilib </em>




<h2>Set compiler flags to disable security features</h2>

When you compile the vulnerable program (explain in the next section) with gcc, use the following compiler flags to disable the security features.

<em>-z execstack </em>

Turn off the NX protection to make the stack executable

<em>-fno-stack-proector </em>

Remove StackGuard that detects stack smashing exploitations

<em>-g </em>

Enable the debugging symbols

<h1>Overview</h1>

<strong> </strong>

The goal of the exploitation is to teach you how buffer overflows work. <strong>You must gain a shell by passing a malicious input into a vulnerable program</strong>. The vulnerability takes as input a file named “badfile”. Your job is to create a badfile that results in the vulnerable program producing a shell. Note that you also have a nop sled to make the vulnerability work even if your shellcode moves by a few bytes. At this point in the lab you should already have the following files on your VM:













<h2><em>BOF.c </em></h2>

In BOF.c there is an un-bounded strcpy, which means anything that is not null- terminated will overwrite the buffer boundaries and (hopefully) put some information into the stack that you will design. Your exploit must work with my version of BOF.c (can’t change it to make your code work).










To compile BOF.c, you need to add the compile flags mentioned. Also, use <em>-m32</em> flag to compile your program for a 32 bit system. This will make it easier to work with memory addresses later on since 32 bit systems have shorter memory addresses than 64 bit systems.




<h3>$ gcc -m32 -g -z execstack -fno-stack-protector BOF.c -o BOF</h3>

<em> </em>

<h2><em>testShellCode.c </em></h2>

This program simply lets you test shell code itself. There are a lot of different “shell codes” you can find or create, and this is a good way to see what they do, and if they’ll work for you (on your operating system).

The actual shellcode you are using is simply the assembly version of this C code:




#include &lt;stdio.h&gt; int main( ) { char *name[2]; name[0] = “/bin/sh”; name[1] = NULL;

execve(name[0], name, NULL);

}













<h2><em>createBadfile.c </em></h2>

This program writes out “badfile”, however currently it is just full of nops (no ops). You need to modify it to place your shell code into it and cause the code to jump to the shellcode. The shellcode included already in createBadfile.c (as a char array) does work. You shouldn’t need to modify it, but you’re welcome to.







To compile the testShellCode.c and createBadfile.c, you can run the following commands. Note that test testShellCode.c requires the <em>-m32 -z execstack</em> flags but createBadfile.c does not.

<h1>Starting the Exploitation</h1>

There are really two challenges in the lab. To execute the shellcode you want to overwrite the return address in the <em>bufferOverflow() </em>function. You must make the return address of that function point to your shellcode.

<ol>

 <li>You need to figure out what memory address the return address is stored in.</li>

 <li>Then you need to figure out the address of your shellcode in memory, and write the shellcode’s address into the return address you found in step 1. In the lab instruction, I will give you some hints for the step 1.</li>

</ol>




<h2>Finding Return Address on the Stack</h2>

In order to find the return address on stacks, we first use GDB, The GNU Project Debugger, to take a look at the assembly code. You can find more information about GDB from here: <u>https://www.gnu.org/software/gdb/</u> Note that you can also use tool, <em>objdump, </em>to read the assembly code.







To run BOF in debugging mode with GDB:




<h3>$ gdb BOF</h3>




<em> </em>

First, we disassemble the main() function of the BOF program. We find the bufferOverflow() function in the main() function (type disas main in the GDB). Then, we disassemble the bufferOverflow() function, which has a vulnerability in it.

<h3>$ (gdb) disas main $ (gdb) disas bufferOverflow</h3>

<em> </em>

<em> </em>

<em> </em>

You need to understand the assembly code to find where the return address is on the stack. Next, type run in the GDB to execute the BOF program.

<h3>$ (gdb) run</h3>

<em> </em>

As we expected, the BOF program generates an exception, segmentation fault. The Instruction Pointer (EIP) is 0x90909090. This is because we put NOP sleds on the badfile that overflows the buffer in the BOF program.

You also can see more register information by execute info register in the GDB

<h3>$ (gdb) info register</h3>

<em> </em>

Note that you can always type help in the GDB to learn the commands.

<h1>Assignments for the Lab 7</h1>

A zip file containing:

<ol>

 <li>Your updated createBadfile.c that generates the input for the BOF program</li>

 <li>A copy of the badfile. This must generate a shell when BOF runs from the command line in the VM</li>

 <li>A screenshot of using BOF program to gain a shell (see simple screenshot below). Add your screenshot at the top of your PDF file.</li>

 <li>A PDF file with answers to the following questions:

  <ol>

   <li>What happens when you compile without “-z execstack”?</li>

   <li>What happens if you enable ASLR? Does the return address change?</li>

   <li>Does the address of the buffer[] in memory change when you run BOF using GDB, /<em>root/Desktop/&lt;your-specific-path-to-BOF&gt;/BOF</em>, and <em>./BOF</em>?</li>

  </ol></li>

</ol>




<strong>Happy Exploiting! </strong>

<strong> </strong>

<strong> </strong>