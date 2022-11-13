# Pythia简要介绍与使用

[toc]

## 1.pythia简介

[PYTHIA](https://pythia.org/)是一个用于产生高能物理碰撞事件的程序，即用于描述电子、质子、光子和重核之间的高能碰撞。它包含了许多物理方面的理论和模型，包括硬相互作用和软相互作用、粒子分布、初始和最终状态pa粒子簇射、多子相互作用、碎裂和衰变。它主要基于原始研究，但也借用了文献中的许多公式和其他知识。因此，它被归类为通用蒙特卡罗事件生成器。

## 2.pythia基本安装

```sh
./configure
make
```

`gmake clean`将移除编译相关的库文件。

## 3.使用案例

`examples`文件提供了学习相关的案例，不调用外部库时，只需要在`example`文件夹下运行

```sh
make mainXX
```

例如，`make main35`

将见到如下c++的编译命令：

```sh
g++ main35.cc -o main35 -I../include -O2 -std=c++11 -pedantic -W -Wall -Wshadow -fPIC -pthread  -L../lib -Wl,-rpath,../lib -lpythia8 -ldl
```

`-o output `

`-I include `

`-L library`

`-O2` Optimize even more.  GCC performs nearly all supported optimizations that do not involve a space-speed tradeoff.  As compared to -O, this option increases both compilation time and the performance of the generated code.

`-pedantic` Issue all the warnings demanded by strict ISO C and ISO C++; reject all programs that use forbidden extensions, and some other programs that do not follow ISO C and ISO C++.  For ISO C, follows the version of the ISO C standard specified by any -std option used.

`-W`  Inhibit all warning messages.

`-Wall` 开启一些警告

`-Wshadow` Warn whenever a local variable or type declaration shadows another variable, parameter, type, class member (in C++), or instance variable (in Objective-C) or whenever a built-in function is shadowed. Note that in C++, the compiler warns if a local variable shadows an explicit typedef, but not if it shadows a struct/class/enum.  Same as -Wshadow=global.

 `-fPIC` If supported for the target machine, emit position-independent code, suitable for dynamic linking and avoiding any limit on the size of the global offset table.  This option makes a difference on AArch64, m68k, PowerPC and SPARC.

`-pthread` Define additional macros required for using the POSIX threads library.  You should use this option consistently for both compilation and linking.  This option is supported on GNU/Linux targets, most other Unix derivatives, and also on x86 Cygwin and MinGW targets.

`-Wl,option`  Pass option as an option to the linker.  If option contains commas, it is split into multiple options at the commas.  You can use this syntax to pass an argument to the option.  For example, -Wl,-Map,output.map passes -Map output.map to the linker.  When using the GNU linker, you can also get the same effect with -Wl,-Map=output.map.




运行并输出结果：

```sh
./main35 >output.dat
```

## 4.源码解析

[粒子PDG编号](./资料/montercarlorpp.pdf)

```c++
#include <iostream>
#include "Pythia8/Pythia.h"

int main()
{
    int nevents =10;
    Pythia8::Pythia pythia;
    
    pythia.readString("Beams:idA = 2212");
    pythia.readString("Beams:idB = 2212");
    pythia.readString("Beams:eCM = 14.e3");//electronic center of mass
    pythia.readString("SoftQCD:all = on");
    pythia.readString("HardQCD:all = on");
    
    Pythia8::Hist hpz("Momentum Distribution",100,-10,10);
    
    pythia.init();
    
    for(int i=0; i<nevents; i++)
    {
        if(!pythia.next())continue;
        
        int entries = pythia.event.size();
		std::cout<< "Event:" <<i<<std::endl;
        std::cout<< "Event size:"<<entries<<std::endl;
        
        for(int j=0;j<entries;j++)
        {
            int id = pythia.event[j].id();
            double m = pythia.event[j].m();
            double px =pythia.event[j].px();
            double py =pythia.event[j].py();
            double pz =pythia.event[j].pz();
            double pabs = sqrt(pow(px,2)+pow(py,2)+pow(pz,2));
            
            hpz.fill(pz);
            std::cout<<id<<" "<<m<<" "<<pabs<<std::endl;
        }
    }
    std::cout<<hpz<<std::endl;
    
    Pythia8::HistPlot hpl("nameofObject");
    hpl.frame("nameofpdf","nameofHistogram","nameofXaxis","nameofYaxis");
    hpl.add(hpz);
    hpl.plot();
    
    return 0;
}       
```

编译一系列案例的`makefile`制作

```makefi
tutorial%: tutorial%.cc
g++ $@.cc -o $@ -I/pathtopythia8/include -O2 -std=c++11 -pedantic -W -Wall -Wshadow -fPIC -pthread  -L/pathtopythia8/lib -Wl,-rpath,/pathtopythia8/lib -lpythia8 -ldl

```

输出文件数据处理

```python
from matplotlib import pyplot as plt
from matplotlib.backends.backend_pdf import PdfPages
pp = PdfPages('output.pdf')
tmp1 = plt.figure(1)
tmp1.set_size_inches(8.00,600)
plot=open('output.dat')
plot = [line.split() for line in plot]
valx = [float(x[0]) for x in plot]
valy = [float(x[1]) for x in plot]
vale = [float(x[2]) for x in plot]
plt.hist(valx,vale,weights=valy,histtype='step',label=r'Momentum Distribution')
plt.xlim(-1e1,1e1)
plt.ylim(0,1.124e2)
plt.ticklabel_format(axis='y',style='sci',scilimits=(-2,3))
plt.legend(frameon=False,loc='best')
plt.title("Momentum Distribution")
plt.xlabel("Momentum")
plt.ylabel('Entries')
pp.savefig(tmp1)
plt.clf()
pp.close()
```

## 5.pythia倒入Root库

makefile变动

```makefile
tutorial%: tutorial%.cc
g++  `root-config --cflags` $@.cc -o $@ -I/pathtopythia8/include -O2 -std=c++11 -pedantic -W -Wall -Wshadow -fPIC -pthread  -L/pathtopythia8/lib -Wl,-rpath,/pathtopythia8/lib -lpythia8 -ldl `root-config --glibs`

```

c++源文档的变动

```c++
#include "TFile.h"
#include "TTree.h"

#include <iostream>
#include "Pythia8/Pythia.h"

int main()
{
    TFile *output = new TFile("out.root","recreate");
    
    TTree *tree = new TTree("tree","tree");
    
    int id,event,size,no;
    double m,px,py,pz;
    
    tree->Branch("event",&event,"event/I");
    tree->Branch("size",&size,"size/I");
    tree->Branch("no",&no,"no/I");
    tree->Branch("id",&id,"id/I");
    tree->Branch("m",&m,"m/D");
    tree->Branch("px",&px,"px/D");
    tree->Branch("py",&py,"py/D");
    tree->Branch("pz",&pz,"pz/D");
    
    int nevents =1e4;
    Pythia8::Pythia pythia;
    
    pythia.readString("Beams:idA = 2212");
    pythia.readString("Beams:idB = 2212");
    pythia.readString("Beams:eCM = 14.e3");//electronic center of mass
    pythia.readString("SoftQCD:all = on");
    pythia.readString("HardQCD:all = on");
    
    Pythia8::Hist hpz("Momentum Distribution",100,-10,10);
    
    pythia.init();
    
    for(int i=0; i<nevents; i++)
    {
        if(!pythia.next())continue;
        
        int entries = pythia.event.size();
		std::cout<< "Event:" <<i<<std::endl;
        std::cout<< "Event size:"<<entries<<std::endl;
        
        for(int j=0;j<entries;j++)
        {
            int id = pythia.event[j].id();
            double m = pythia.event[j].m();
            double px =pythia.event[j].px();
            double py =pythia.event[j].py();
            double pz =pythia.event[j].pz();
            double pabs = sqrt(pow(px,2)+pow(py,2)+pow(pz,2));
            
            hpz.fill(pz);
            std::cout<<id<<" "<<m<<" "<<pabs<<std::endl;
            
            tree->Fill();
        }
    }
    output->Write();
    output->Close();
    
    std::cout<<hpz<<std::endl;
    
    Pythia8::HistPlot hpl("nameofObject");
    hpl.frame("nameofpdf","nameofHistogram","nameofXaxis","nameofYaxis");
    hpl.add(hpz);
    hpl.plot();
    
    return 0;
}       
```

## 6.Pythia多线程的使用

```c++
#include "TFile.h"
#include "TTree.h"

#include "TThread.h"

#include <iostream>
#include "Pythia8/Pythia.h"

void *handle(void *ptr)
{
    int ith = (long)ptr;
    
    TFile *output = new TFile("out.root","recreate");
    
    TTree *tree = new TTree("tree","tree");
    
    int id,event,size,no;
    double m,px,py,pz;
    
    tree->Branch("event",&event,"event/I");
    tree->Branch("size",&size,"size/I");
    tree->Branch("no",&no,"no/I");
    tree->Branch("id",&id,"id/I");
    tree->Branch("m",&m,"m/D");
    tree->Branch("px",&px,"px/D");
    tree->Branch("py",&py,"py/D");
    tree->Branch("pz",&pz,"pz/D");
    
    int nevents =1e4;
    Pythia8::Pythia pythia;
    
    pythia.readString("Beams:idA = 2212");
    pythia.readString("Beams:idB = 2212");
    pythia.readString("Beams:eCM = 14.e3");//electronic center of mass
    pythia.readString("SoftQCD:all = on");
    pythia.readString("HardQCD:all = on");
    
    pythia.init();
    
    for(int i=0; i<nevents; i++)
    {
        if(!pythia.next())continue;
        
        int entries = pythia.event.size();
		std::cout<< "Event:" <<i<<std::endl;
        std::cout<< "Event size:"<<entries<<std::endl;
        
        for(int j=0;j<entries;j++)
        {
            int id = pythia.event[j].id();
            double m = pythia.event[j].m();
            double px =pythia.event[j].px();
            double py =pythia.event[j].py();
            double pz =pythia.event[j].pz();
            double pabs = sqrt(pow(px,2)+pow(py,2)+pow(pz,2));
                   
            tree->Fill();
        }
    }
    output->Write();
    output->Close();
}

int main()
{
 	TThread *th[4];
    for(int i=0;i<4;i++)
    {
    	th[i]=new TThread(Form("th%d",i),handle,(void*)i);
        th[i]->Run();
    }
    
    for(int i=0;i<4;i++)
    {
        th[i]->Join();
    }
    return 0;
}      
```

## 参考资料

[1] pythia.org

[2] youtube教程
