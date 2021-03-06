---
title: 浅析android手游lua脚本的加密与解密
date: 2018-07-07 16:01:12
tags: 
  - lua
  - luajit
  - 手游安全
categories: lua加解密
---
# 前言
&emsp;&emsp;博客刚刚弄完善，把去年发在看雪的一篇精华帖转了过来，文章稍微修改了下，并且增加了后续[文章](https://litna.top/2018/07/08/浅析android手游lua脚本的加密与解密（后续）/)，希望能够吸引点人气。这篇文章是我在学习android手游安全时总结的一篇关于lua的文章，不足之处欢迎指正，也欢迎各位大佬前来交流。

&emsp;&emsp;主要用到的工具和环境：

> * win7系统
> * cocos2d-x 开发环境
> * IDA6.8
> * vs2015
> * AndroidKiller 1.3.1
> * luadec51
> * luajit-decomp
> ...

<!-- more -->

# lua 现状分析
&emsp;&emsp;去年的那篇文章这一章没有写的，今年补上了一篇lua加解密的相关工作，请看：[《浅析android手游lua脚本的加密与解密（前传）》](https://litna.top/2018/07/07/浅析android手游lua脚本的加密与解密（前传）/)

# lua 各文件关系
&emsp;&emsp;在学习lua手游解密过程中，遇到的lua文件不外乎就3种。其中.lua后缀的文件是明文代码，直接用记事本就能打开，.luac是lua脚本编译后的字节码文件，文件头为0x1B 0x4C 0x75 0x61，lua虚拟机能够直接解析lua和luac脚本文件，而.luaJIT是另一个lua的实现版本（不是lua的原作者写的），JIT是指Just-In-Time（即时解析运行），luaJIT相比lua和luac更加高效，文件头是0x1B 0x4C 0x4A：

&emsp;&emsp;luac 文件头如下：

{% asset_img luac.png %}

&emsp;&emsp;luaJIT 文件头如下：

{% asset_img luajit.png %}

# lua 脚本保护
&emsp;&emsp;一般有安全意识的游戏厂商都不会直接把lua源码脚本打包到APK中发布，所以一般对lua脚本的保护有下面3种：

**1. 普通的对称加密，在加载脚本之前解密**

&emsp;&emsp;这种情况是指打包在APK中的lua代码是加密过的，程序在加载lua脚本时解密（加载脚本的关键函数luaL_loadbuffer），解密后就能够获取lua源码。如果解密后获取的是luac字节码的话，也可以通过对应的反编译得到lua源码，反编译主要用的工具有unluac和luadec51，后面会具体分析。

**2. 将lua脚本编译成luaJIT字节码并且加密打包**

&emsp;&emsp;因为cocos2d-x使用的luaJIT，而且luaJIT反编译后的结果阅读起来比较麻烦，所以这种情况能够较好的保护lua源码。这个情况主要是先解密后反编译，反编译主要是通过luajit-decomp项目，它能够将luajit字节码反编译成伪lua代码。

**3. 修改lua虚拟机中opcode的顺序**

&emsp;&emsp;这种情况主要是修改lua虚拟机源码，再通过修改过的虚拟机将lua脚本编译成luac字节码，达到保护的目的。这种情况如果直接用上面的反编译工具是不能将luac反编译的，需要在虚拟机的引擎中分析出相对应的opcode，然后修复反编译工具luadec 源码中的 opcode 并重新编译，编译后的文件就能进行反编译了，后面会具体分析。

&emsp;&emsp;在破解手游过程中，上面的三种情况可能会交叉遇到。

# 获取lua源码的一般方法
&emsp;&emsp;这里主要介绍4种方法，都会在后面用实例说明。

**1. 静态分析so解密方法**

&emsp;&emsp;这种方法需要把解密的过程全部分析出来，比较费时费力，主要是通过ida定位到luaL_loadbuffer函数，然后往上回溯，分析出解密的过程。

**2. 动态调试：ida + idc + dump**

&emsp;&emsp;游戏会在启动的时候通过调用 luaL_loadbuffer函数加载必要的lua脚本，我们可以通过ida动态调试so文件，然后是定位到luaL_loadbuffer地址，再下断点 ，断下后就接着运行idc脚本（或者python脚本）将lua代码导出（程序调用一次luaL_loadbuffer只加载一个lua脚本，所以需要编写idc脚本自动保存lua代码）。

**3. hook so**

&emsp;&emsp;跟4.2原理一样，就是通过hook函数luaL_loadbuffer地址，将lua代码保存，相比4.2的好处是有些lua脚本需要在玩游戏的过程中才加载，如果用了4.2的方法，那么在游戏过程中需要加载新的lua文件就会中断一次，我们就需要手动运行一次idc脚本，如果是hook的话，就不需要那么麻烦，直接玩一遍游戏，全部lua脚本就已经保存好了。

**4. 分析lua虚拟机的opcode的顺序**

&emsp;&emsp;这里主要是opcode的顺序被修改了，需要用ida定位到虚拟机执行luac字节码的地方，然后对比原来lua虚拟机的执行过程，获取修改后的opcode顺序，最后还原lua脚本。

&emsp;&emsp;综上，静态分析费时费力但是能够解密全部的lua脚本，而通过动态获取的方法虽然方便，但是只能获取游戏当前需要加载的lua脚本。具体选择哪种方法，需要衡量时间成本等。

# lua脚本解密实例分析
&emsp;&emsp;接着用3个游戏作为实例说明上面分析的情况。

## 54捕鱼
&emsp;&emsp;首先用AndroidKiller 加载，然后查看lib目录下的so文件，发现libcocos2dlua.so文件，基本可以确定是lua脚本编写的了。这里有个小技巧，当有很多so文件的时候，一般最大的文件是我们的目标（文件大是因为集成了lua引擎，既然有lua引擎，那么肯定有lua脚本了）。接着找lua脚本，资源文件和lua脚本文件都是在assets目录下。我们发现这个游戏的资源文件和配置文件都是明文，这里直接修改游戏的配置文件就可以作弊（比如修改升级炮台所需的金币和钻石，就可以达到快速升级炮台的目的），然后并没有发现类似lua脚本的文件。

&emsp;&emsp;顺手解压了一下res目录下的liveupdate_precompiled.zip，发现解压失败，看来是加密了（看文件名字知道是更新游戏的代码）这里说明一下，一般遇到xxxx_precompiled.zip的这种文件，都是quick-cocos2d-x框架（quick简单来说就是对lua的拓展实现），在quick-cocos2d-x框架下可以用compile_scripts命令将lua文件加密打包成xxxx_precompiled.zip，游戏运行时再解密加载。注意，这种方式打包的lua脚本一般都会被编译成luaJIT字节码，加载的关键函数是loadChunksFromZIP，可以在IDA中直接搜索该函数，如果找不到可以搜索字符串luaLoadChunksFromZIP来定位到函数

&emsp;&emsp;OK，了解了原理接下来开始动手分析，将libcocos2dlua.so拖到IDA中加载，函数中直接搜索loadChunksFromZIP，定位后F5分析。

{% asset_img 5.1_1.png %}

&emsp;&emsp;对该函数一直向上回溯（交叉引用 ），来到下图，发现解密的密钥和签名，其中xiaoxian为密钥，XXFISH为签名

{% asset_img 5.1_2.png %}

&emsp;&emsp;进去函数里面看看，其实会发现调用了XXTea算法，这里我们也可以直接分析loadChunksFromZIP函数的源码（所以配置一个cocos2d的开发环境还是非常有必要的）。查看源码里的lua_loadChunksFromZIP函数的原型：
```c++
int CCLuaStack::lua_loadChunksFromZIP(lua_State *L)
{
    if (lua_gettop(L) < 1)
    {     // 这里可以发现用字符串也可以定位到目标函数
        CCLOG("lua_loadChunksFromZIP() - invalid arguments");
        return 0;
    }

...
        if (isXXTEA)
        {
            // decrypt XXTEA
            // 这里调用了解密函数
            xxtea_long len = 0;
            buffer = xxtea_decrypt(zipFileData + stack->m_xxteaSignLen,
                                   (xxtea_long)size - (xxtea_long)stack->m_xxteaSignLen,
                                   (unsigned char*)stack->m_xxteaKey,
                                   (xxtea_long)stack->m_xxteaKeyLen,
                                   &len);
            delete []zipFileData;
            zipFileData = NULL;
            zip = CCZipFile::createWithBuffer(buffer, len);
        }
...
}

```

&emsp;&emsp;接下来直接写解密函数（在cocos2d-x项目里面写的解密函数，很多工具类直接可以调用）
```c++
void decryptZipFile_54BY(string strZipFilePath)
{
        CCFileUtils *utils = CCFileUtils::sharedFileUtils();

        unsigned long lZipFileSize = 0;
        unsigned char *szBuffer = NULL;
        unsigned char *zipFileData = utils->getFileData(strZipFilePath.c_str(), "rb", &lZipFileSize);

        xxtea_long xxBufferLen = 0;
        szBuffer = xxtea_decrypt(zipFileData + 6,           //6为签名XXFISH的长度
               (xxtea_long)lZipFileSize - (xxtea_long)6,    //减去签名的长度
               (unsigned char*)"xiaoxian",                  //xiaoxian为密钥
               (xxtea_long)8,                               //密钥的长度
               &xxBufferLen);

        //获取zip里面的所有文件
        CCZipFile *zipFile = CCZipFile::createWithBuffer(szBuffer, xxBufferLen);
        int count = 0;
        string strFileName = zipFile->getFirstFilename();
        while (strFileName.length())
        {
               cout << "filename:" << strFileName << endl;

               unsigned long lFileBufferSize = 0;
               unsigned char *szFileBuffer = zipFile->getFileData(strFileName.c_str(), &lFileBufferSize);

               if (lFileBufferSize)
               {
                       ++count;

                       ofstream ffout(strFileName, ios::binary);
                       ffout.write((char *)szFileBuffer, sizeof(char) * (lFileBufferSize));
                       ffout.close();

                       delete[] szFileBuffer;
               }
               strFileName = zipFile->getNextFilename();
        }
        delete[] zipFileData;
}
```
&emsp;&emsp;解密后的文件如下：

{% asset_img 5.1_3.png %}

&emsp;&emsp;这几个都是更新游戏的代码，是luajit的文件，所以接下来需要反编译。反编译需要确定lua和luajit的版本，我们通过IDA查看下lua版本和luajit版本，字符串中分别搜索lua+空格和luajit+空格：

&emsp;&emsp;lua版本为5.1

{% asset_img 5.1_4.png %}

&emsp;&emsp;luajit版本为2.1.0

{% asset_img 5.1_5.png %}

&emsp;&emsp;这篇文章反编译用到的是luajit-decomp，这里需要注意，luajit-decomp默认的lua 5.1 和luajit 2.0.2，我们需要下载对应lua和luajit的版本，编译后替换luajit-decomp下的lua51.dll、luajit.exe、jit文件夹。反编译时需要替换的文件和文件夹如下：

{% asset_img 5.1_6.png %}

&emsp;&emsp;对于这个游戏，我们需要下载版本为2.1.0-beta2的luajit，并且编译生成文件后，复制LuaJIT-2.1.0-beta2\src路径下的lua51.dll、luajit.exe文件和jit文件夹覆盖到luajit-decomp目录中。luajit-decomp用的是autolt3语言，原脚本默认是只反编译当前目录下的test.lua文件，所以需要修改decoder.au3文件的代码。修改后的代码另存为jitdecomp.au3文件，接着编译au3代码为jitdecomp.exe。我这里还增加了data目录，该目录下有3个文件夹，分别为：

    luajit：待反编译的luajit文件
    asm：反汇编后的中间结果
    out：反编译后的结果

&emsp;&emsp;将解密后的文件放到luajit文件夹，运行 jitdecomp.exe，反编译的结果在out目录下，结果如下：

{% asset_img 5.1_7.png %}

&emsp;&emsp;这个反编译工具写得并不好，反编译后的lua文件阅读起来相对比较困难，而且反编译的lua格式有问题，所以不能用lua编辑器格式化代码。

## 捕鱼达人4
&emsp;&emsp;这个游戏主要是用ida动态调试so文件，然后用idc脚本把lua文件全部dump下来的方法。首先用AndroidKiller加载apk，在lib目录下有3个文件夹，不同的手机cpu型号对应不同的文件夹 。本人的手机加载的目标so文件在armeabi-v7a文件下：

{% asset_img 5.2_1.png %}

&emsp;&emsp;接着，ida加载libcocos2dlua.so文件，定位到函数luaL_loadbuffer，可以在函数中直接搜索，也可以字符串搜索 "[LUA ERROR]" 来定位到函数中，函数分析如下：

```c++
LUALIB_API int luaL_loadbuffer (lua_State *L, const char *buff, size_t size,const char *name)
```

&emsp;&emsp;所以在ARM汇编中，参数R0为lua_State指针，参数R1为脚本内容，R2为脚本大小，R3为脚本的名称，写一段IDC脚本dump数据即可：

```c++
#include <idc.idc>

static main()
{
    auto code, bp_addrese,fp,strPath,strFileName;

    bp_addrese = 0x7573022C;                                                // luaL_loadbuffer函数地址
    AddBpt(bp_addrese);                                                     // 下断点，也可以手动下断

    while(1)
    {
        code = GetDebuggerEvent(WFNE_SUSP|WFNE_CONT, 15);                   // 等待断点发生，等待时间为15秒
        if ( code <= 0 )
        {
            Warning("错误代码：%d",code);
            return 0;
        }

        Message ("地址：%a, 事件id：%x\n", GetEventEa(), GetEventId());      // 断点发生，打印消息
        strFileName = GetString(GetRegValue("R3"),-1,0);                    // 获取文件路径名
        strFileName = substr(strFileName,strrstr(strFileName,"/")+1,-1);    // 获取最后一个‘/’后面的名字（文件的名字）去掉路径
        strPath = sprintf("c:\\lua\\%s",strFileName);                       // 保存lua的本地路径

        fp = fopen(strPath,"wb");
        savefile(fp,0,GetRegValue("R1"),GetRegValue("R2"));
        fclose(fp);

        Message("保存文件成功: %s\n",strPath);
    }
}


//字符串查找函数，从后面向前查找，返回第一次查找的字符串下标
static strrstr(str,substr1)
{
    auto i,index;
    index = -1;
    while (1)
    {
        i = strstr(str,substr1);
        if (-1 == i) return index;
        str = substr(str,i+1,-1);
        index = index+i+1;
    };
}
```

&emsp;&emsp;ida动态调试so文件网上有很多文章，这里就不详细说明了。通过idc脚本获取的部分数据如下：

{% asset_img 5.2_2.png %}

&emsp;&emsp;虽然文件的后缀名是.luac，但其实都是明文的lua脚本。

## 梦幻西游手游
&emsp;&emsp;AndroidKiller反编译apk，查看lib下存在libcocos2dlua.so，基本上确定是lua写的：

{% asset_img 5.3_1.png %}

&emsp;&emsp;在assets\HashRes目录下，存在很多被加密的文件，这里存放的是lua脚本和游戏的其他资源文件：

{% asset_img 5.3_2.png %}

&emsp;&emsp;接着找lua脚本的解密过程，用ida加载libcocos2dlua.so文件，搜索luaL_loadbuffer函数，定位到关键位置，这里就是解密的过程了：

{% asset_img 5.3_3.png %}

&emsp;&emsp;分析解密lua文件过程如下：

{% asset_img 5.3_4.png %}

&emsp;&emsp;这里需要实现Lrc4解密的相关函数，还有Lzma解压函数需要自己实现，其他几个都是cocos2d平台自带的函数，直接调用就可以了。上面的流程图实现的函数如下：

```c++
bool decryptLua_Mhxy(string strFilePath, string strSaveDir)
{
        bool bResult = false;
        char *szBuffer = NULL;
        int nBufferSize = 0;

        CCFileUtils *utils = CCFileUtils::sharedFileUtils();

        unsigned long ulFileSize = 0;
        char *szFileData = (char*)utils->getFileData(strFilePath.c_str(), "rb", &ulFileSize);

        if (strncmp(szFileData, "L:grxx", 6))
        {
               if (!strncmp(szFileData, "__sign_of_g18_enc__", 0x13))
               {
                       szBuffer = szFileData + 0x13;
                       nBufferSize = ulFileSize - 0x13;

                       bResult = decrypt((unsigned char*)szBuffer, nBufferSize);
               }
        }
        else if (!strncmp(szFileData + 6, "__sign_of_g18_enc__", 0x13))
        {
               unsigned char *pData = (unsigned char *)szFileData + 0x19;
               int nLen = ulFileSize - 0x19;

               bResult = decrypt(pData, nLen);

               if (ZipUtils::isGZipBuffer(pData, nLen))
               {
                       nBufferSize = ZipUtils::ccInflateMemory(pData, nLen, (unsigned char**)&szBuffer);
               }
               else if (ZipUtils::isCCZBuffer(pData, nLen))
               {
                       nBufferSize = ZipUtils::inflateCCZBuffer(pData, nLen, (unsigned char**)&szBuffer);
               }
               else if (LzmaUtils::isLzmaBuffer(pData, nLen))
               {
                       nBufferSize = LzmaUtils::inflateLzmaBuffer(pData, nLen, (unsigned char**)&szBuffer);
               }
               else
               {
                       bResult = false;
               }
        }

        if(bResult)
               saveLuaData(szBuffer, nBufferSize, strSaveDir);

        return bResult;
}
```

&emsp;&emsp;解密函数过程如下：

{% asset_img 5.3_5.png %}

&emsp;&emsp;decrypt()实现代码如下：

```c++
bool decrypt(unsigned char *pData, int nLen)
{
        Lrc4 *pLrc4 = new Lrc4;
        Lrc4_lrc4(pLrc4);
        Lrc4_s(pLrc4, pData, nLen);
        return true;
}

```
&emsp;&emsp;Lrc4结构如下:

```c++
#define DATA_SIZE 256

struct Lrc4
{
        unsigned char pData[DATA_SIZE];  //初始化时计算得到的256个字节
        int nIndex;                      //记录下标
        int nPreIndex;                   //记录前一个下标
};

```

&emsp;&emsp;其他函数的具体实现请看DecryptData_Mhxy.cpp文件，这里就不贴代码了。解密后的文件如下：

{% asset_img 5.3_6.png %}

&emsp;&emsp;可以看出，解密后的文件为luac字节码，但是这里直接用反编译工具是不能反编译luac字节码的，因为游戏的opcode被修改过了，我们需要找到游戏opcode的顺序，然后生成一个对应opcode的luadec.exe文件才能反编译。下表为修改前后的opcode：

{% asset_img 5.3_7.png %}

&emsp;&emsp;lua虚拟机的相关内容就不说明了，百度很多，这里说明下如何还原opcode的顺序。首先需要定位到opmode的地方，IDA搜索字符串"LOADK"，定位到opname的地方，交叉引用到代码，找到opmode：

{% asset_img 5.3_8.png %}

&emsp;&emsp;off_B02CEC为opname的地址，byte_A67C00为opmode的地址，进入opmode地址查看：

{% asset_img 5.3_9.png %}

&emsp;&emsp;这里没有把全部数据截图出来，可以看出，这里的opmode跟原opmode是不对应的。原opmode在lua源码中的lopcodes.c文件中：

{% asset_img 5.3_10.png %}

&emsp;&emsp;源码用了宏，计算出来的结果就是上表中opmode的结果。这里对比opmode就可以快速对比出opcode，因为opmode不相等，那么opcode也肯定不相等，到这一步，已经能还原部分opcode了，因为有一些opmode是唯一的。比如下面几个：

{% asset_img 5.3_11.png %}

&emsp;&emsp;如SETLIST，原opcode为34，opmode为0x14，找到的opmode的第8个字节也为0x14，则实际上SETLIST的opcode为8。

&emsp;&emsp;接下来就需要定位到luaV_execute函数，然后对比源码来还原其他的opcode，直接IDA搜索字符串"initial value must be a number"可以定位到luaV_execute 函数，再F5一下。接着打开lua源码中的lvm.c文件，找到luaV_execute函数，就可对比还原了。lua源码和IDA F5后的代码其实差别还是有的，而且源码用了大量的宏，所以源码只是用来参考、理解lua虚拟机的解析过程，本人在还原的过程中，会再打开一个没有修改opcode的libcocos2dlua.so文件，这样对比查找就方便多了。

&emsp;&emsp;最后修改lua源码 lopcodes.h中的opcode、lopcodes.c的opname和opmode，重新编译并生成luadec51 .exe（需要将lua源码中的src目录放到luadec51的lua目录下才能编译），就OK了，写个批处理文件就可以批量反编译。一个文件反编译的结果：

{% asset_img 5.3_12.png %}

# 总结
&emsp;&emsp;总结一下解密lua的流程，拿到APK，首先反编译，查看lib目录下是否有libcocos2dlua.so，存在的话很大可能这个游戏就是lua编写，其中lib目录下文件最大的就是目标so文件，一般情况就是libcocos2dlua.so。接着再看assert文件夹有没有可疑的文件，手游的资源文件会放到这个文件夹下，包括lua脚本。其次分析lua加密的方式并选择解密脚本的方式，如果可以ida动态调试，一般都会选择用idc脚本dump下lua代码。最后如果得到的不是lua明文，还需要再反编译一下。

&emsp;&emsp;不足之处：luajit的反编译并不完美，用的是luajit-decomp反编译工具，工具的作者也说只是满足了他自己的需求，还有一个luajit反编译是python写的工具ljd。其次梦幻luac的反编译部分代码反编译失败，修复过程请看这篇[文章](https://litna.top/2018/07/08/梦幻手游部分Luac反编译失败的解决方法/)。

# 参考文章
* 腾讯游戏安全中心 《Lua游戏逆向及破解方法介绍》 http://gslab.qq.com/portal.php?mod=view&aid=173
* 云风 《Lua源码欣赏》 https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/luadec/云风-lua源码欣赏-lua-5.2.pdf
* FSD-BlueEffie的博客 《梦幻西游手游 美术资源加密分析》 http://blog.csdn.net/blueeffie/article/details/50971665
* Kaitiren的专栏 《Quick-cocos2d-x 与Cocos2dx 区别》 http://blog.csdn.net/kaitiren/article/details/35276177