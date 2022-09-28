# 如何在Linux下設定錄音筆時間[^1]

買了一個錄音筆，效果比使用筆記本話筒錄音好多了還省電。當然啦，我也曾試過使用手機錄音，結果是，沒能錄多久就中斷了（Android 就是這麼不靠譜）。

我的錄音需要記錄較為準確的時間資訊。錄音筆怎麼知道現在是什麼時間呢？還好它沒有跟風，用不著聯網！

它帶了一個小程式，叫「錄音筆專用時間同步工具」（英文叫「SetUDiskTime」，可以搜到的）。是一個 EXE 檔案，以及一個 DLL 檔案。功能很棒，沒有廣告，沒有推薦，也不需要註冊什麼亂七八糟的賬戶，甚至都不需要開啟瀏覽器訪問人家官網。就彈一個框，顯示當前時間，確定一下就設定好時間了。這年頭，這麼單純的 Windows 軟體還真是難得呢。

然而，它不支援我用的 Linux 啊。雖然我努力地保證這錄音筆一直有電，但是時間還是丟失了幾次，它的FAT檔案系統也髒了幾次。每次我都得開 WinXP 虛擬機器來設定時間，好麻煩。

Wine 是不行的，硬體相關的東西基本上沒戲。拿 Procmon 跟蹤了一下，也沒什麼複雜的操作，主要部分就幾個 DeviceIoControl 呼叫，但是看不到呼叫引數。試了試 IDA，基本看不懂……不過倒是能知道，它通過 IOCTL_SCSI_PASSTHROUGH 直接給裝置傳送了 SCSI 命令。

既然跟蹤不到，試試抓 USB 的包好了。本來想用 Wireshark 的，但是 WinXP 版的 Wireshark 看來不支援。又嘗試了裝置分配給 VBox 然後在 Linux 上抓包，結果 permission denied……我是 root 啊都被 deny 了……

那麼，還是在 Windows 上抓包吧。有一個軟體叫 USBPcap，下載安裝最新版，結果遇到 bug。那試試舊版本吧。官網沒給出舊版本的下載地址，不過看到下載連結帶上了版本號，這就好辦了。去 commit log 裡找到舊的版本號替換進去，https://dl.bintray.com/desowin/USBPcap/USBPcapSetup-1.0.0.7.exe，就好了～

抓好包，取到 Linux 下扔給 Wireshark 解讀。挺小的呢，不到50個包，大部分還都是重複的。很快就定位到關鍵位置了：

一個 0xcc 命令發過去，裝置回覆「ACTIONSUSBD」，大概是讓裝置做好準備。然後一個 0xb0 命令，帶上7位元組資料發過去，時間就設定好了。簡單明瞭，不像那些小米空氣淨化器之類的所謂「物聯網」，通訊加密起來不讓人好好使用。

那麼，這7位元組是怎麼傳遞時間資料的呢？我首先檢查了UNIX時間戳，對不上。後來傳送這個字串看上去挺像YYYYMMDDHHMMSS格式的，只是明顯不是當時的時間。啊，它是十六進位制的嘛！心算了幾個，符合！再拿出我的 Python 牌計算器，確定年份是小端序的16位整數。

好了，協議細節都弄清楚了，接下來是實現。我原以為我得寫個 C 程式，調幾個 ioctl 的，後來網友說有個 sg3_utils 包。甚好，直接拿來用 Python 調，省得研究那幾個 ioctl 要怎麼寫。

```python
#!/usr/bin/env python3
import os
import sys
import struct
import subprocess
import datetime
def set_time(dev):
 cmd = ['sg_raw', '-s', '7', dev, 'b0', '00', '00', '00', '00', '00',
   '00', '07', '00', '00', '00', '00']
 p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stderr=subprocess.PIPE)
 dt = datetime.datetime.now()
 data = struct.pack('<HBBBBB', dt.year, dt.month, dt.day,
      dt.hour, dt.minute, dt.second)
 _, stderr = p.communicate(data)
 ret = p.wait()
 if ret != 0:
 raise subprocess.CalledProcessError(ret, cmd, stderr=stderr)
def actionsusbd(dev):
 cmd = ['sg_raw', '-r', '11', dev, 'cc', '00', '00', '00', '00', '00',
   '00', '0b', '00', '00', '00', '00']
 subprocess.run(cmd, check=True, stderr=subprocess.PIPE)
def main():
 if len(sys.argv) != 2:
 sys.exit('usage: setudisktime DEV')
 dev = sys.argv[1]
 if not os.access(dev, os.R_OK | os.W_OK):
 sys.exit(f'insufficient permission for {dev}')
 actionsusbd(dev)
 set_time(dev)
if __name__ == '__main__':
 main()
```

[^1]: Copied from https://codertw.com/%E4%BC%BA%E6%9C%8D%E5%99%A8/376108
