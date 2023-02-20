
## 背景
这是一个奇怪而少见的场景：我工作在 A 网络中，需要 B 网络中的 Server 给我提共服务（编译中的一个步骤），但是 A 和 B 是不可以相互连接的。而 A 和 B 网络现有的可用工具只有 ftp。所以常规的流程就是：
1. 在 A 网络中生成所需文件；
2. 将其通过 ftp 发送至一个中转网络的电脑上；
3. 中转电脑（C网络中）将文件通过 ftp 发送到 B 网络的电脑上；
4. B 网络中的电脑，处理完，将生成的文件通过 ftp 发送到中转电脑；
5. 中转电脑将文件通过 ftp 发送到 A 网络的电脑上；

## 设计
实际应用中非常浪费时间，并且浪费人力。所以就想到用 ftplib 来自动化处理这些事务。设计如下：

![](attachments/Pasted%20image%2020230220113939.png)

## Code

### A 网络端
```python
import hashlib
import os
import sys
import ftplib
import configparser
import time

config = configparser.ConfigParser()
config.read('local.ini')
ftp_work_dir = config['dir']['ftp']
ip = config['net']['ip']
username = config['net']['username']
password = config['net']['password']

def file_hash(file_path: str, hash_method) -> str:
    if not os.path.isfile(file_path):
        print('文件不存在。')
        return ''
    h = hash_method()
    with open(file_path, 'rb') as f:
        while b := f.read(8192):
            h.update(b)
    return h.hexdigest()

def file_sha256(file_path: str) -> str:
    return file_hash(file_path, hashlib.sha256)

def download_file(ftp, remote_file, localfile):
    with open(localfile, 'wb') as fp:
        print(ftp.retrbinary('RETR ' + remote_file, fp.write))
    fp.close() 

def uploadfile(ftp, remoteFile, localFile):
    with open(localFile, 'rb') as fp:
        print(ftp.storbinary('STOR ' + remoteFile, fp, 1024))
    fp.close() 

if __name__ == "__main__":
    ftp = ftplib.FTP_TLS()
    print(ftp.connect(ip, 21))
    print(ftp.login(username, password))
    print(ftp.prot_p())
    print(ftp.cwd(ftp_work_dir))

    if len(sys.argv) == 2:
        codesign_file = sys.argv[1]
        flag_file = file_sha256(codesign_file)
        os.system('echo ' + codesign_file + '___' + flag_file[0:8] + '>' + flag_file[0:8])
        uploadfile(ftp, 'codesign/' + codesign_file + '___' + flag_file[0:8], codesign_file)
        uploadfile(ftp, 'flag/' + flag_file[0:8], flag_file[0:8])

        print('please wait ......')
        while not 'done__' + flag_file[0:8] in ftp.nlst('./'):
            time.sleep(2)
        bin_zip = 'ABC_bin___' + codesign_file.split('.7z')[0] + '__' + flag_file[0:8] + '.zip'
        download_file(ftp, 'bin/' + bin_zip, bin_zip)
        ftp.delete('bin/' + bin_zip)
        ftp.delete('done__' + flag_file[0:8])
        os.remove(flag_file[0:8])        
    else: 
        print('argument error!')

    ftp.close()
```
有个配置文件，如此程序的通用性更好：
```toml
[dir]
ftp=ftp_work_dir

[net]
ip=ftp_ip
username=ftp_username
password=ftp_password
```

### 中转端
```python
import os
import ftplib
import time
import configparser

codesign_hash = ''

config = configparser.ConfigParser()
config.read('transfer.ini')
ftp_work_dir = config['dir']['ftp']
A_ip = config['A_net']['ip']
A_username = config['A_net']['username']
A_password = config['A_net']['password']
B_ip = config['B_net']['ip']
B_username = config['B_net']['username']
B_password = config['B_net']['password']

def download_file(ftp, remote_file, localfile):
    with open(localfile, 'wb') as fp:
        print(ftp.retrbinary('RETR ' + remote_file, fp.write))
    fp.close() 

def uploadfile(ftp, remoteFile, localFile):
    with open(localFile, 'rb') as fp:
        print(ftp.storbinary('STOR ' + remoteFile, fp, 1024))
    fp.close() 

def monitor_A_ftp(A_ftp, B_ftp):
    global codesign_hash
    res = None
    codesigns_flag = A_ftp.nlst('flag')
    while len(codesigns_flag) == 0:
        codesigns_flag = A_ftp.nlst('flag')
        time.sleep(2)
    flag = codesigns_flag[0].split('/')[1]
    download_file(A_ftp, codesigns_flag[0], flag)
    with open(flag, 'r') as f:
        res = f.readline().split('\n')[0]  # res means codesign files
        print(res)
    if res != None:
        codesign_hash = 'done__' + res.split('___')[1]
        download_file(A_ftp, 'codesign/' + res, res)
        uploadfile(B_ftp, 'codesign/' + res, res)
        uploadfile(B_ftp, codesigns_flag[0], flag)
        A_ftp.delete('codesign/' + res)
        A_ftp.delete(codesigns_flag[0])
        os.remove(res)
        os.remove(flag)

def monitor_B_ftp(A_ftp, B_ftp):
    global codesign_hash
    print('please wait ......')
    while not codesign_hash in B_ftp.nlst('./'):
        time.sleep(2)
    abc_bin = B_ftp.nlst('bin')[0]
    local_bin = abc_bin.split('/')[1]
    download_file(B_ftp, codesign_hash, codesign_hash)
    download_file(B_ftp, abc_bin, local_bin)
    uploadfile(A_ftp, abc_bin, local_bin)
    uploadfile(A_ftp, codesign_hash, codesign_hash)
    B_ftp.delete(abc_bin)
    B_ftp.delete(codesign_hash)
    os.remove(local_bin)
    os.remove(codesign_hash)

def while_monitor(A_ftp, B_ftp):
    while True:
        monitor_A_ftp(A_ftp, B_ftp)
        monitor_B_ftp(A_ftp, B_ftp)

if __name__ == "__main__":
    A_ftp = ftplib.FTP_TLS()
    print(A_ftp.connect(A_ip, 21))
    print(A_ftp.login(A_username, A_password))
    print(A_ftp.prot_p())
    print(A_ftp.cwd(ftp_work_dir))
    
    B_ftp = ftplib.FTP_TLS()
    print(B_ftp.connect(B_ip, 21))
    print(B_ftp.login(B_username, B_password))
    print(B_ftp.prot_p())
    print(B_ftp.cwd(ftp_work_dir))

    while_monitor(A_ftp, B_ftp)

    A_ftp.close()
    B_ftp.close()
```
同样有个配置文件：
```toml
[dir]
ftp=ftp_work_dir

[A_net]
ip=ftp_ip
username=ftp_username
password=ftp_password

[B_net]
ip=ftp_ip
username=ftp_username
password=ftp_password
```

### B 网络端
```python
import sys
import os
import ftplib
import time
import subprocess
import shutil
import configparser

config = configparser.ConfigParser()
config.read('remote.ini')
ftp_work_dir = config['dir']['ftp']
ip = config['net']['ip']
username = config['net']['username']
password = config['net']['password']

def download_file(ftp, remote_file, localfile):
    with open(localfile, 'wb') as fp:
        print(ftp.retrbinary('RETR ' + remote_file, fp.write))
    fp.close() 

def uploadfile(ftp, remoteFile, localFile):
    with open(localFile, 'rb') as fp:
        print(ftp.storbinary('STOR ' + remoteFile, fp, 1024))
    fp.close() 

def run_codesign(codesign_dir):
    shutil.copyfile('001_run.bat', codesign_dir + '/001_run.bat')
    shutil.copyfile('002_move.bat', codesign_dir + '/002_move.bat')
    os.remove(codesign_dir + '/CodeSign.exe')
    os.remove(codesign_dir + '/CodeSign_Account.bat')
    shutil.copyfile('CodeSign.exe', codesign_dir + '/CodeSign.exe')
    shutil.copyfile('CodeSign_Account.bat', codesign_dir + '/CodeSign_Account.bat')
    p = subprocess.Popen('cmd.exe /c 001_run.bat', cwd=codesign_dir)
    stdout, stderr = p.communicate()
    print(p.returncode)
    p = subprocess.Popen('cmd.exe /c 002_move.bat', cwd=codesign_dir)
    stdout, stderr = p.communicate()
    print(p.returncode)

def monitor_tw_ftp(tw_ftp):
    res = None
    codesigns_flag = tw_ftp.nlst('flag')
    print(codesigns_flag)
    if len(codesigns_flag) != 0:
        flag = codesigns_flag[0].split('/')[1]
        download_file(tw_ftp, codesigns_flag[0], flag)
        with open(flag, 'r') as f:
            res = f.readline().split('\n')[0]
            print(res)
        if res != None:
            download_file(tw_ftp, 'codesign/' + res, res)
            codesign_dir = res.split('___')[0].split('.7z')[0]
            cmd = '"C:/Program Files/7-Zip/7z.exe" x ' + res + ' ' + codesign_dir
            os.system(cmd)
            bin_zip = 'ABC_bin___' + codesign_dir + '__' + res.split('___')[1] + '.zip'
            if os.path.exists(codesign_dir + '/a_all.bin'):
                run_codesign(codesign_dir)
                cmd = '"C:/Program Files/7-Zip/7z.exe" a ' + bin_zip + ' ' + codesign_dir + '/ABC_bin'
                os.system(cmd)
            else:
                subdir = os.listdir(codesign_dir)[0]
                run_codesign(codesign_dir + '/' + subdir)
                cmd = '"C:/Program Files/7-Zip/7z.exe" a ' + bin_zip + ' ' + codesign_dir + '/' + subdir + '/ABC_bin'
                os.system(cmd)
            uploadfile(tw_ftp, 'bin/' + bin_zip, bin_zip)
            os.system('echo done > done__' + res.split('___')[1])
            uploadfile(tw_ftp, 'done__' + res.split('___')[1], 'done__' + res.split('___')[1])
            tw_ftp.delete('codesign/' + res)
            tw_ftp.delete(codesigns_flag[0])
            # os.removedirs(codesign_dir)
            shutil.rmtree(codesign_dir, ignore_errors=True)
            os.remove(bin_zip)
            os.remove('done__' + res.split('___')[1])
            os.remove(flag)
            os.remove(res)

if __name__ == "__main__":
    ftp = ftplib.FTP_TLS()
    print(ftp.connect(ip, 21))
    print(ftp.login(username, password))
    print(ftp.prot_p())
    print(ftp.cwd(ftp_work_dir))

    while True:
        monitor_tw_ftp(ftp)
        time.sleep(2)
```
配置文件如下：
```toml
[dir]
ftp=ftp_work_dir

[net]
ip=ftp_ip
username=ftp_username
password=ftp_password
```

## 使用

完成后，`work_dir` 下便有了一个 `zip` 文件（包含结果文件）。
```bash
PS C:\work_dir> py .\local.py SourceFile.7z
220 Welcome to Phison FTP2
230 使用者 Hosinglobal 登入
200 Protection set to Private
250 CWD 命令成功執行
226 傳送完畢
226 傳送完畢
please wait ......
226 傳送完畢
PS C:\work_dir>
```