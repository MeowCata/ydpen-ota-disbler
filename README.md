# ydpen-ota-disbler
前几天在b站看到有大佬成功破解了有道新版词典笔的adb（[教程](https://www.bilibili.com/read/cv40931661)），火树入群后花20找管理破了adb，还是花钱能吃细糠，管理在kali里一个脚本就给adb整的明明白白😋

打开adb后装了 [PenTool](https://github.com/penosext/miniapp)，支持ai和一堆小功能，好的，词典笔已经化为在学校最强的生产力了

但是看到同学词典笔更新系统时想到，补兑！以网易一向的尿性，破adb的人这么多肯定要采取措施，到时候甩出patch把当前adb破解方法ban了可就不好说了（我相信大佬很快就会找出新方法的，但多一事不如少一事嘛）。因此，**本文来对有道词典笔一些可用tweak研究研究**

> [!TIP]
> 作者使用词典笔型号 `A6 Pro`，系统版本 `4.7.4`，已破adb
> 
> 使用 AI： Gemini 3 Thinking

## Block OTA Update
> [!WARNING]
> 严格遵循本教程可以得到预想的结果。作者 **不对使用该教程中任何部分所产生的 任何后果 承担责任**（包括但不限于：有道官方警告 家长管控 词典笔功能损坏 等）

Gemini 给出的思路是，先找到进程：
```bash
ps -w
```
我的部分输出：
```bash
 1378 root       756 S    /usr/bin/guardian_run /usr/bin/runOtaMgr
 1380 root      1504 S    {runOtaMgr} /bin/sh /usr/bin/runOtaMgr

 1428 root     14480 S    /usr/bin/ota_mgr
```
所以可以看到控制OTA更新的无非两个，`runOtaMgr` & `ota_mgr`，杀掉就行了...

吗？

`kill -9 <PID>` 后来到词典笔 设置-系统更新 界面，发现还能正常检查更新，说明两个进程没死透，被守护进程重新拉起了

而由于词典笔是 `Read-only` 系统，读写操作受限，因此修改 `/bin/` `/etc/`等文件夹里面文件的操作就不要想了

此时我们可以去找找可以读写的目录挂载，执行 `mount`，部分输出：
```bash
[root@YoudaoDictionaryPen-732:/oem/YoudaoDictPen/output]# mount

/dev/mmcblk0p10 on /userdata type ext4 (rw,relatime)
/dev/mmcblk0p11 on /userdisk type ext4 (rw,relatime)
/dev/mmcblk0p10 on /etc/input-event-daemon.conf type ext4 (rw,relatime)
```
好！现在我们知道了 `userdata` `userdisk` `input-event-daemon.conf` 都是支持 `rw` 的，那可以考虑 `Bind Mount`，手动把我们要写的杀掉OTA更新的脚本挂载上去

开始干吧，首先在 `userdata` 文件夹创建 `skip_ota.sh` 并让它啥也不干：
```bash
echo "#!/bin/sh" > /userdata/skip_ota.sh
echo "exit 0" >> /userdata/skip_ota.sh
chmod +x /userdata/skip_ota.sh
```
然后挂到OTA更新上面去：
```bash
mount --bind /userdata/skip_ota.sh /usr/bin/ota_mgr
mount --bind /userdata/skip_ota.sh /usr/bin/runOtaMgr
```
这时我们再次尝试杀掉OTA进程，它再次被拉起时就会执行我们写的空脚本，这就实现了禁用OTA更新
```bash
kill -9 1378 1380
kill -9 1428
# 对应上面的PID
```
再看看检查更新，好诶！一直卡在了“正在检测更新...”界面，说明我们的方法奏效了

但是毕竟挂载是存在内存里的，重启一下就没了。有没有什么能开机自启的办法呢？

突然想到，管理在破adb的时候在 `/userdisk/skip_re` 中写的 `skip_login.sh` 绕过adb密码脚本是随开机自启的，理论上可以通过修改这个脚本实现开机附带把OTA更新也杀了。但考虑到不是人人找的都是一个管理破adb，方法不尽相同，所以来个通用版的吧

Gemini 考虑到有道词典笔基于 `Linux` 内核，**"系统在内核加载完后，会按照顺序执行 /etc/init.d/ 下所有以大写字母 S 开头的脚本。数字越小执行越早，数字越大执行越晚"**，并且上面执行 `mount` 时明确看到了 `input-event-daemon.conf` 是映射到 `/userdata/` 的，Gemini 的原话：
> 在一个只读的 EROFS 系统里，这种“狸猫换太子”的挂载动作不可能凭空发生，它必然是写在某个启动脚本里的。既然目标文件在 /etc 下，那负责挂载它的脚本百分之百就躲在 /etc/init.d/ 里面

到 `/etc/init.d/` 一看：
```bash
[root@YoudaoDictionaryPen-732:/userdata]# ls /etc/init.d/

S01DisableDebugUart    S01systeminit          S20blocklink           S20network             S20urandom             S21dhcpcd              S21mountall.sh         S22-app-launcher       S22syslogd             S23klogd               S30mkswap              S40load_modules        S50launcher            S80dnsmasq             S99_run_test_scripts   S99collect_panic       S99crond               S99input-event-daemon  ninfod.sh              rcK                rcS
```
哇哦哦，这个叫 **`S99_run_test_script`** 的文件就是我们要找的，用 cat 一看内容：
```bash
#! /bin/sh

[ -f /userdisk/skip_re/skip_login.sh ] || exit 0

start() {
        printf "Starting test script: skip_login.sh "
    /userdisk/skip_re/skip_login.sh &
        echo "done"
}


stop() {
        printf "Stopping test script: skip_login.sh "
        killall skip_login.sh
        echo "done"
}

restart() {
        stop
        start
}

# See how we were called.
case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  restart|reload)
        restart
        ;;
  *)
        echo "Usage: $0 {start|stop|reload|restart}"
        exit 1
esac

exit $?
```
完全胜利！我们已经找到了控制其他程序开启自启的脚本，任意一根破了adb的词典笔都能通过修改上述文件实现开机运行自定义脚本

直接执行：
```bash
mkdir -p /userdisk/skip_re
cat > /userdisk/skip_re/skip_login.sh <<EOF
#!/bin/sh
mount --bind /userdata/skip_ota.sh /usr/bin/ota_mgr
mount --bind /userdata/skip_ota.sh /usr/bin/runOtaMgr
pkill -f runOtaMgr
killall -9 ota_mgr
touch /tmp/.adb_auth_verified
touch /userdata/boot_success_log
EOF
```

然后赋予权限：
```bash
chmod +x /userdisk/skip_re/skip_login.sh
dos2unix /userdisk/skip_re/skip_login.sh
```
最后 `reboot` 即可体验没有OTA更新的快乐 这才是自由的词典笔！

## Make Your Pen Enduring
> [!WARNING]
> 本部分涉及到 CPU 降频(*Downclocking*)，**可能导致轻微卡顿与性能下降**，但 **可以延长续航时间**。请权衡考虑并使用

执行 `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies` 可以得到 CPU 支持的频率，使用 `watch -n 1 cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq` 可以监视当前 CPU 频率。观察发现 CPU 频率在 `[600MHz <-> 816MHz <-> 1.104GHz <-> 1.6GHz]` 区间内跳动

可以查看可用的 CPU 频率调整模式：
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
# return: <ondemand> <userplace> <performance>
```
* **ondemand**: 按需模式，默认使用这个，频率不定，功耗平衡
* **userplace**: 锁定特定频率（不推荐，调高了耗电，调低了卡顿，而且档位数太少）
* **performance**: 高频率运行，真的很流畅，真的很耗电

综上，我应用了比较平衡的调整，*ondemand 模式 + 锁定最高频率到 816Mhz (默认最高 1.6Ghz)* 
```bash
echo ondemand > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# 限制最高频 816MHz
echo 816000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```
总体测下来感觉还好，没有比原先卡很多，但电池续航一定会更长
<img width="350" height="85" alt="image" src="https://github.com/user-attachments/assets/7f5bb9d2-688c-4eb6-8956-0e06eff582a4" />

⬆️该图测于词典笔连接adb，不运行任何应用

上图可能暗示了有道词典笔广泛吐槽点的问题所在：熄屏听歌不怎么耗电，反而放一边待机耗电量非常可观。这可能因为息屏时 CPU 频率维持在中位数值，从而耗电，而听歌所带来的 CPU 消耗非常小，让频率反而到了更低的数值？以上说法待核实

## Block Logs Transferring
> [!WARNING]
> 已知：执行以下更改会导致 **语音助手，语音输入，单词本同步，写作指导，AI语法分析等 一系列系统应用无法联网**，请权衡考虑
>
> 也许会有定向阻止记录日志的方法，那才是从根源解决问题的路子（still 研究中）
>
> 由此可见，仅仅通过阻止特定ip而禁用某些功能是不可行的，最上面说的停用OTA更新的方法才是真的香

看到远古项目 [PenMod](https://github.com/PenUniverse/PenMods)有防日志上传机制，考虑到新版词典笔会保存所有的 `UserAction` 到 `\userdata\applog`，为了保护隐私，可选择禁用上传日志（包括自动上传）：

非常简单，执行 `netstat` 看当前所有的连接，发现 `117.135.207.132` 这个ip十分可疑，不管那么多先ban了（我猜它就是日志上传的服务器ip）：
```bash
route add -net 117.135.207.0 netmask 255.255.255.0 reject 2>/dev/null
```
可拦截该ip所有网段，如果你哪天不忍心了就执行下面这个命令删掉：
```bash
route del -net 117.135.207.0 netmask 255.255.255.0 reject
```
通过 `route -n` 查看当前路由表（类似于管理路由规则）

执行完后可以在终端试着 `ping 117.135.207.132` ，显示 `ping: connect: No route to host` 你就成功了

也可以在词典笔 设置-关于-日志上报 试着上报一下，连着网却显示上报失败说明你也成功啦

以上说的是一次性方案，也想开机自启的话在上面文件末尾添加：
```bash
route add -net 117.135.207.0 netmask 255.255.255.0 reject 2>/dev/null
ifconfig wlan0 down
ifconfig wlan0 up
```
