Linux 源码文件太多了，整体导入 source insight 会非常卡慢，Sync 也需要很久。且不同硬件平台又有很多相同的 symbol，需要阅读的时候眼动区分太累了。这个方法就可以精准导入你的平台所需的源码文件。
### 一、下载所需文件
Linux 源代码：
``` 
https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.1.tar.xz
```
Source Insight 4 安装文件：
```
https://assets-sourceinsight.sfo2.digitaloceanspaces.com/v4/release/sourceinsight40148_7177-setup.exe
```
Source Insight 4 免费使用的 patch:
```
https://github.com/YukiIsait/SourceInsight4Patch/releases/download/v1.0.1/msimg32.dll
```

### 二、安装 source insght 4
运行安装 source insight 4，并在安装后将 dll 文件放到 sourceinsight4.exe 同级文件夹中。即可正常使用。
![](Pasted%20image%2020260101230411.png)

### 三、编译 Linux 内核源码（wsl2 Ubuntu24.04）
安装编译 Linux 内核所需编译器（我编译是 RISC-V 平台，根据自己的平台选择相应的编译器）
``` bash
sudo apt install -y gcc-riscv64-linux-gnu
```
安装编译 Linux 内核所需环境
``` bash
sudo apt install -y build-essential libncurses-dev flex bison openssl   libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf
```
解压 Linux 源码压缩包
``` bash
tar Jxvf linux-6.18.1.tar.xz
```
配置编译 config
``` bash
make ARCH=riscv defconfig
```
编译（将打印重定向到文件中，后续有用）
``` bash
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- all -j8 > build_log.txt
```
编译时间可能需要十几分钟，由电脑性能决定。期间可以用以下指令查看编译进度
``` bash
cat build_log.txt | tail
```

### 四、解析 Build Log
编译时我们来准备解析编译 Log 提取文件列表的脚本。可以用 vim 创建脚本
```
vim sg.sh
```
将以下脚本粘贴进去，注意 `ABS_PATH` 配置为 Windows 路径，但是斜杠用 Linux 的正斜杠，脚本最后会替换。
```bash
#!/bin/sh
ARCH=arm
MACH=imx
FILE_IN=$1
FILE_OUT=$2
#windows abs path
ABS_PATH="D:/Codes/C/linux-6.18.1/"
# .c
SOURCE_LIST=""
# generated file list
FILE_LIST=""
# nest depth for function get_includes()
NEST_DTPTH=0
# recursive function, used to get included files from files.
# result is stored in FILE_LIST
# $1 : file list, e.g. "fs/ext4/file.c fs/ext4/fsync.c"
get_includes()
{
        local includes
        local file
        for file in $1
        do
                if [ ! -e ${file} ]; then
                        continue
                fi
                if echo "${FILE_LIST}" | grep -E ${file} > /dev/null; then
                        continue
                fi
                FILE_LIST="${FILE_LIST} ${file}"
                NEST_DTPTH=$((NEST_DTPTH+1))
                echo "<${NEST_DTPTH} : ${file}"
                includes=$(                                                                                \
                        grep -E -H '^#include' ${file} |                                \
                        sed -r \
                                -e 's@^.*<(acpi/.*)>@include/\1@'                 \
                                -e 's@^.*<(asm-generic/.*)>@include/\1@'\
                                -e 's@^.*<(config/.*)>@include/\1@'         \
                                -e 's@^.*<(crypto/.*)>@include/\1@'         \
                                -e 's@^.*<(drm/.*)>@include/\1@'                 \
                                -e 's@^.*<(generated/.*)>@include/\1@'         \
                                -e 's@^.*<(keys/.*)>@include/\1@'                 \
                                -e 's@^.*<(linux/.*)>@include/\1@'                 \
                                -e 's@^.*<(math-emu/.*)>@include/\1@'         \
                                -e 's@^.*<(media/.*)>@include/\1@'                 \
                                -e 's@^.*<(misc/.*)>@include/\1@'                 \
                                -e 's@^.*<(mtd/.*)>@include/\1@'                 \
                                -e 's@^.*<(net/.*)>@include/\1@'                 \
                                -e 's@^.*<(pcmcia/.*)>@include/\1@'         \
                                -e 's@^.*<(rdma/.*)>@include/\1@'                 \
                                -e 's@^.*<(rxrpc/.*)>@include/\1@'                 \
                                -e 's@^.*<(scsi/.*)>@include/\1@'                 \
                                -e 's@^.*<(sound/.*)>@include/\1@'                 \
                                -e 's@^.*<(target/.*)>@include/\1@'         \
                                -e 's@^.*<(trace/.*)>@include/\1@'                 \
                                -e 's@^.*<(uapi/.*)>@include/\1@'                 \
                                -e 's@^.*<(video/.*)>@include/\1@'                 \
                                -e 's@^.*<(xen/.*)>@include/\1@'                 \
                                -e "s@^.*<(asm/.*)>@arch/${ARCH}/include/\1 arch/${ARCH}/include/generated/\1@"        \
                                -e "s@^.*<(mach/.*)>@arch/${ARCH}/mach-${MACH}/include/\1@"        \
                                -e 's@(^.*/)[^/]+\.c.*\"(.*)\"@\1\2@'         \
                                -e 's@/\*.*@@'                                                         \
                                -e 's@^.*\#include.*$@@'                                  \
                                -e 's@^@ @' |                                                        \
                        sort |                                                                                 \
                        uniq |                                                                                \
                        tr -d '\n' |                                                                 \
                        tr -d '\r'                                                                        \
                )
                if [ -n "${includes}" ]; then
                        get_includes "${includes}"
                fi
                echo ">${NEST_DTPTH}) : ${file}"
                NEST_DTPTH=$((NEST_DTPTH-1))
        done
}
# get *.c from kernel build log
SOURCE_LIST=$(                                                \
        grep -E '^\s*CC' ${FILE_IN} |        \
        sed -r                                                         \
                -e 's/^\s*CC\s*/ /'                        \
                -e 's/\.o/\.c/'                        |        \
        tr -d '\n' |                                         \
        tr -d '\r'                                                \
)
echo ${SOURCE_LIST}
get_includes "${SOURCE_LIST}"
FILE_LIST=$(echo "${FILE_LIST}" | sed -r -e 's/\s/\r\n/g' )
echo "${FILE_LIST}" > ${FILE_OUT}
#sed -i 's///\\/g' ${FILE_OUT}
#替换行首为windows路径
sed -i "s#^#${ABS_PATH}#g" ${FILE_OUT}
#替换linux路径符'/'为windows路径符'\'
sed -i "s#/#\\\#g" ${FILE_OUT}
```
给脚本增加执行权限并解析 build log，输出文件列表到 `files_list.txt` 中。
```
chmod +x sg.sh
./sg.sh build_log.txt files_list.txt
```

### 五、用 files_list 导入文件 Linux 源码到 source insight
打开 source insight 4 并创建工程，之后依次点击 Progect → Add and Remove Project Files... → Add from list... → files_list.txt → 打开，即可将编译过程中涉及到的文件都导入 source insight 了。
![](Pasted%20image%2020260101231325.png)
再点击 Synchronize Files... 去建立 symbol 映射关系，之后便可以愉快阅读了。
![](Pasted%20image%2020260101230659.png)