# 向ELF文件中嵌入代码
## 要求
在ELF可执行文件中添加新功能，保证不破坏本来的程序，不得不认真思考如何添加代码，添加到什么地方，要修改那些头部数据，如何修改，修改成什么值，都需要熟悉ELF格式，并用合适的方法，找到合适的工具，来完成这些修改。
网上有一篇文章提供了基本的思路，写一段包含elf.h头文件的C程序，来完成这些修改，这篇文章并没有提供详细的教程，我们只能根据资料一步一步摸索，反复调试，一起讨论，才能解决问题。
## 摘要
可执行链接格式(Executable and Linking Format)是应用程序二进制接口的一部分，是一种可移植的目标文件格式；ELF目标文件有三种类型：可重定位文件，可执行文件和共享目标文件，本报告描述如何修改一个现有的ELF可执行程序，使得该可执行程序能够实现新的功能而不影响原文件的正常执行以及功能。我们小组受到一篇文章的启发，引用C的elf.h头文件，此头文件定义了Elf头部，程序头部等相关结构的结构体，定义了结构体中内容的类型，我们以此可以通过C程序读取原执行文件，然后利用头文件中定义的结构体准确修改需要变动的数据，嵌入新的机器码，再一并写入到新的可执行文件中去。若要达到往ELF可执行文件中添加新功能的目的，先要了解ELF可执行文件，比如可执行文件有哪几个部分组成，各个部分涵盖什么内容，内容之间的联系等等；因为修改是基于Linux，所以要求能够熟练使用Linux Shell常用命令与和elf相关几个命令，比如readelf查看elf文件信息，objdump能反汇编执行程序；能自主使用gcc，gdb等编译调试工具；添加新功能时，只能操作机器码，无法直接修改高级语言源码，所以需要了解汇编码与机器码。
## 功能需求
1.	能完整读取ELF可执行文件。
2.	识别ELF可执行文件的各个部分。
3.	能分辨可执行文件各个部分的每一个数据。
4.	在当前创建一个新文件，若存在，则将原文件中存在的所有数据抹掉。
5.	对原文件ELF头部表，程序头部表，节区头部表进行修改使其符合嵌入代码后的新文件。
6.	写一段在根目录创建文件并写入 “Hello”最后跳转的汇编程序，并将代码转换到机器码。
7.	能把ELF所有头部表和新的机器码一并写入新文件，使新文件能正确执行

## 设计思路

| ELF头部 |
| :----:|
| 程序头部表  |
|  段1  |
|  段2    |
|    ...    |
|  节区头部表    |

  ELF可执行程序分为四个部分，文件开始处是一个ELF头部(ELF Header)，用来描述整个文件的组织。节区部分包含链接视图的大量信息，指令、数据、符号表、重定位信息等等。
  
  程序头部表(Program Header Table)，告诉系统如何创建进程映像。用来构造进程映像的目标文件必须具有程序头部表。
  
  节区(Sections)，包含目标文件中的所有信息。
  
  节区头部表(Section Header Table)包含了描述文件节区的信息，每个节区在表中都有一项，每一项给出诸如节区名称，节区大小这类信息。
大致了解了ELF执行文件，可见其文件是一个整体，层层紧扣，若要添加一串代码以实现新功能，必须考虑大局。首先，我们考虑使用objdump工具直接反汇编，然后在这个基础，直接对可执行程序的机器码进行更改，添加目标代码，然后我们将vim编辑器反汇编，发现其代码可阅读性极其差，根本无法定位我们需要修改的数据位置，从哪里添加目标码也无从下手，而且换一个其他的ELF可执行文件，又要做复杂庞大的重复工作，于是否决了这个思路。
就在一筹莫展之时，我们在网上发现，有一个叫elf.h的头文件，其中定义了ELF头部，程序头部，节区头部的结构体和结构体中变量的类型，根据elf执行文件的结构顺序是ELF头部、程序头部，节区、节区头部，我们写一个C语言程序，包含elf.h头文件，依此读取elf结构的各个部分，就可以让ELF可执行文件变得可操作化。

  首先要考虑的是，我们应该将我们要嵌入的目标码插入到文件中哪一部分呢；通过对ELF执行文件的了解，代码和数据是在节区之中，而相邻的几个节区又组成一个程序段，我们要找的节区，必须要在一个可执行的程序段，所以我们要嵌入的程序段应该是程序执行入口所在的程序段而为了嵌入目标码之后要改动的数据尽量小，而且不影响该可执行段的正常功能，所以本段中的最后一个节区是最好的选择。
  
  利用elf.h头文件中的结构体，我们可以先读取ELF头部，获取程序入口e_entry，节区头部表偏移量e_shoff等信息；接着读取程序头表，通过e_entry与程序头表中程序段的虚拟地址信息，定位出需要的可执行段，借此获取可执行段的虚拟地址与大小，之后跳过节区，读取节区头部，利用获取的可执行段的虚拟地址与大小，找到该段的最后一个节区，并记录下该节区的虚拟地址，以此，就可以定位到我们需要嵌入代码的位置。
	
  下一步，可以着手于往新的文件中写入所有的信息；与前面步骤一样，依次读取程序的各个部分，修改ELF头部的程序入口虚拟地址，使其指向我们将要添加新代码的位置，增加节区头部表的偏移量，增加程序头表里我们将要插入代码的段的大小，增加该段后面每一段的偏移量，接着写入程序头表与节区头表的所有数据，并嵌入我们要添加的机器码，最后修改该节区的大小，并增加之后每一个节区的偏移量。将所有数据写入即可得到一个新的Elf可执行文件。

## 开发环境、安装和配置
- 系统：Fedora_32bit
下载地址：
> https://download-ib01.fedoraproject.org/pub/fedora-secondary/releases/28/Workstation/i386/iso/Fedora-Workstation-Live-i386-28-1.1.iso

- 前期环境准备
> yum install gcc  
> yum install git

- 下载源码
> git clone https://gist.github.com/923797ce0244609adbb6040b426eb223.git

> cd 923797ce0244609adbb6040b426eb223

- 编译源文件
> gcc ./injectelf.c –o ./injectelf

- 嵌入代码
> ./injectelf originFileName newFileName

嵌入gcc为例
> ./injectelf /usr/bin/gcc /usr/bin/newgcc   

此时此时newgcc在编译之前会先在根目录创建testfile并写入Hello。
注意，在输入originFile之前，先输入命令 
> readelf –a originFileName | less

检测是否是32位可执行文件类型

## 代码解读
读取ELF头部，并记录程序原入口地址
```C
char elf_ehdr[sizeof(Elf32_Ehdr)];
Elf32_Ehdr *p_ehdr;
p_ehdr = (Elf32_Ehdr *)elf_ehdr;
int origfile = open(argv[1], O_RDONLY);
int ret = read(origfile, elf_ehdr, sizeof(elf_ehdr));
Elf32_Addr orgi_entry = p_ehdr->e_entry;
```
读取程序头部，找到程序入口所在的程序段。方法是循环读取每个程序头部，寻找段开始的虚拟地址在orgi_entry之前而且段结束的虚拟地址在orgi_entry之后。
```C
char elf_phdr[sizeof(Elf32_Phdr)];
Elf32_Phdr *p_phdr; 
Elf32_Addr program_head_vaddr;
Elf32_Word program_head_sizes;
p_phdr = (Elf32_Phdr *)elf_phdr;
for (int i = 0; i < (int)p_ehdr->e_phnum; i++) {
	read(origfile, elf_phdr, sizeof(elf_phdr));
	if (p_phdr->p_paddr < orgi_entry && (p_phdr->p_paddr + p_phdr->p_filesz)>orgi_entry) {
		program_head_vaddr = p_phdr->p_vaddr;
		program_head_sizes = p_phdr->p_filesz;
	}
}
```
跳过节区头部表与程序头部表之间的部分，循环读取节区头部表，找到该程序段最后一个节区，记录下该节区的地址、偏移量与大小， 并将该节区末尾记录为new_entry变量。

完成读取，开始写入。
```C
char parasize[] = {
	'H', 'e', 'l', 'l', 'o', '\0', //file content
	'/', 't', 'e', 's', 't', 'f', 'i', 'l', 'e', '\0', //file name
	0x50, //push eax (offset 0, size 1
	0x53, //push ebx (offset 1, size 1
	0x51, //push ecx (offset 2, size 1
	0x52, //push edx (offset 3, size 1
	0xb8, 0x05, 0x00, 0x00, 0x00, //mov eax, 5 (offset 4, size 5
	0xbb, 0x00, 0x00, 0x00, 0x00, //mov ebx, 0 (offset 9, size 5
	0xb9, 0x00, 0x00, 0x00, 0x00, //mov ecx, 0 (offset 14, size 5
	0xba, 0x09, 0x03, 0x00, 0x00, //mov edx, 777 (offset 19, size 5
	0xcd, 0x80, //int 0x80 (offset 24, size 2
	0x89, 0xc3, //mov ebx, eax (offset 26, size 2
	0xb9, 0x00, 0x00, 0x00, 0x00, //mov ecx, 0 (offset 28, size 5
        0xba, 0x00, 0x00, 0x00, 0x00, //mov edx, 0 (offset 33, size 5
	0xb8, 0x04, 0x00, 0x00, 0x00, //mov eax, 4 (offset 38, size 5
	0xcd, 0x80, //int 0x80 (offset 43, size 2
	0xb8, 0x06, 0x00, 0x00, 0x00, //mov eax, 6 (offset 45, size 5
	0xcd, 0x80, //int 0x80 (offset 50, size 2
	0x5a, //pop edx (offset 52, size 1
	0x59, //pop ecx (offset 53, size 1
	0x5b, //pop ebx (offset 54, size 1
	0x58, //pop eax (offset 55, size 1
	0xbd, 0x00, 0x00, 0x00, 0x00, //mov ebp, 0 (offset 56, size 5
	0xff, 0xe5, //jmp ebp (offset 57, size 2
};
```
首先定义了要插入的数据和机器码的字符串。该机器码首先将eax、ebx、ecx、edx入栈，再调用了linux的三个syscall，分别是打开文件、写文件、关闭文件，再依次出栈edx、ecx、ebx、eax。  

然后借由strlen函数得到定义的写文件的字符串的长度和文件目录的字符串长度，得到新的入口地址（跳过数据段从可执行的机器码开始执行）。

```C
p_ehdr->e_entry = new_entry + precodeSize;
p_ehdr->e_shoff += 4096;
write(newfile, elf_ehdr, sizeof(elf_ehdr));
```
改了原来elf头部的入口地址后，重新将该结构体写入新文件。
```C
for(int i = 0; i<(int)p_ehdr->e_phnum; i++){
    read(origfile,elf_phdr,sizeof(elf_phdr));
    if (p_phdr->p_paddr < orgi_entry && (p_phdr->p_paddr + p_phdr->p_filesz)>orgi_entry) {
        p_phdr->p_filesz += 4096;
	p_phdr->p_memsz += 4096;
    }else if(p_phdr->p_offset > entry_section_offset)
        p_phdr->p_offset += 4096;
    write(newfile,elf_phdr,sizeof(elf_phdr));
}
```
接下来一段写程序头部，将可执行段的长度增加一个page size，所有在可执行段后面的程序段偏移量增加一个page size。其余程序头部不处理直接复制原程序头部。

接下来将new_entry之前的节区原样复制。接下来把机器码中相关的偏移量以及调用函数的参数赋值，并把机器码插入到new¬_entry后面。再把原new¬_entry后面的节区原样复制过来。
```C
for(int i=0;i<(int)p_ehdr->e_shnum;i++){
	read(origfile,elf_shdr,sizeof(elf_shdr));
	if (p_shdr->sh_offset == entry_section_offset) {
		p_shdr->sh_size +=4096;
	}else if(p_shdr->sh_offset > entry_section_offset){
		p_shdr->sh_offset +=4096;
	}
	write(newfile,elf_shdr,sizeof(elf_shdr));
}
```
接下来写节区头部，把所拓展的节区size增加一个page size。所有在所拓展的节区后方的节区偏移量增加一个page size，其余节区头部原样复制。

至此完成所有注入工作。

写下这段程序的难点在于如何通过各个头部的层层联系找到需要嵌入代码的节区，以及如何编写正确的目标码。

在测试程序时，一直segmentation fault，无法运行，经过调试，发现根本没找到目标节区，多次检查，竟然原因是忘记给节区指针赋值，修复之后，可以生成新的执行文件了，但执行报错segmentation fault,ll newfile之后，发现newfile没有比原文件大4096byte，后来检查，经组员讨论，不应该写节区的时候按每个节区头部提供的地址和大小写，节区与节区之间可能还存在symbol table,string table等信息，而是应该将程序头部与节区头部之间的信息整段整段的写入，修改之后，依然segmentation fault,实在是恼人。我们借助gdb调试工具设置断点，逐句调试，发现程序能正常执行，后来负责写汇编的组员才主要没有提前将寄存器入栈，结束后出栈，真是细节决定成败。
