title: msf 终端命令
date: 2014-10-29 21:04
tags:
- kail
- metasploit
categories:
- metasploit
---

- <font color="orange">show exploits</font> 
 列出metasploit框架中的所有渗透攻击模块。
- <font color="orange">show payloads</font>
 列出metasploit框架中的所有攻击载荷。
- <font color="orange">show auxiliary</font>
 列出metasploit框架中的所有辅助攻击载荷。
- <font color="orange">search name</font>
 查找metasploit框架中所有的渗透攻击和其他模块。
- <font color="orange">info</font>
 展示出制定渗透攻击或模块的相关信息。
- <font color="orange">use name</font>
 装载一个渗透攻击或模块。
- <font color="orange">LHOST</font>
 你本地可以让目标主机连接的IP地址，通常当目标主机不在同一个局域网内时，就需要是一个公共IP地址，特别为反弹式shell使用。
- <font color="orange">RHOST  </font>
 远程主机或是目标主机。
- <font color="orange">set function</font>
 设置特定的配置参数（EG：设置本地或远程主机参数）。
- <font color="orange">setg function</font>
 以全局方式设置特定的配置参数（EG：设置本地或远程主机参数）。
- <font color="orange">show options</font>
 列出某个渗透攻击或模块中所有的配置参数。
- <font color="orange">show targets</font>
 列出渗透攻击所有支持的目标平台。
- <font color="orange">set target num</font>
 指定你所知道的目标的操作系统以及补丁版本类型。 
- <font color="orange">set payload name</font>set payload name
 指定想要使用的攻击载荷。
- <font color="orange">show advanced</font>
 列出所有高级配置选项。
- <font color="orange">set autorunscript migrate -f</font>
 在渗透攻击完成后，将自动迁移到另一个进程。
- <font color="orange">check</font>
 检测目标是否选定渗透攻击存在相应的安全漏洞。
- <font color="orange">exploit</font>
 执行渗透攻击或模块来攻击目标。
- <font color="orange">exploit -j</font>
 在计划任务下进行渗透攻击（攻击将在后台进行）。
- <font color="orange">exploit -z</font>
 渗透攻击完成后不与回话进行交互。
- <font color="orange">exploit -e encoder</font>
 制定使用的攻击载荷编码方式（EG：exploit -e shikata_ga_nai）。
- <font color="orange">exploit -h </font>
 列出exploit命令的帮助信息。
- <font color="orange">sessions -l</font>
 列出可用的交互会话（在处理多个shell时使用）。
- <font color="orange">sessions -l -v</font> 
 列出所有可用的交互会话以及详细信息，EG：攻击系统时使用了哪个安全漏洞。
- <font color="orange">sessions -s script</font>
 在所有活跃的metasploit会话中运行一个特定的metasploit脚本。
- <font color="orange">sessions -K</font>
 杀死所有活跃的交互会话。
- <font color="orange">sessions -c cmd</font>
 在所有活跃的metasploit会话上执行一个命令。
- <font color="orange">sessions -u sessionID</font>
 升级一个普通的win32 shell到metasploit shell。
-  <font color="orange">name</font>
 创建一个数据库驱动攻击所要使用的数据库（EG：db_create autopwn）。
-  <font color="orange">db_connect name</font>
 创建并连接一个数据库驱动攻击所要使用的数据库（EG：db_connect user:passwd@ip/sqlname）。
-  <font color="orange">db_namp</font>
 利用nmap并把扫描数据存储到数据库中（支持普通的nmap语句，EG：-sT -v -P0）。
-  <font color="orange">db_autopwn -h</font>
 展示出db_autopwn命令的帮助信息。
-  <font color="orange">db_autopwn -p -r -e</font>  
 对所有发现的开放端口执行db_autopwn，攻击所有系统，并使用一个反弹式shell。
-  <font color="orange">db_destroy</font>
 删除当前数据库。
-  <font color="orange">db_destroy user：passwd@host：port/database</font>
 使用高级选项来删除数据库。
