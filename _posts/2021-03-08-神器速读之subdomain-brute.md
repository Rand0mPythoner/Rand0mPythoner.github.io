---
layout: post
title: "速读神器之subdomain-brute"
subtitle: "WTF about subdomain-brute"
author: "Rand0m ^ Novocaine#p1an0@qq.com"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - Python
  - subdomain-brute
---

0x01 序
--
- 上一篇读神器是hackbar，主要是读了一些功能的实现，也从中学到不少vue和chrome扩展的知识。
- 这一篇打算读一下lijiejie大佬的subdomain-brute，在我阅读过程中觉得可能有阅读障碍的有：
  - dns.resolver
  - Queue(不会也没关系，趁着这个契机了解一下消息队列，不影响理解逻辑)
- 我这个版本的subdomain-brute比较老了，一直没更新过，不知道有什么改动，也不想看新的了，新版有什么比较隐晦的改动想交流可以vx：Rand000m
- 最开始看宫音师傅的带你读神器系列一直觉得应该读核心代码，但是发现很多人代码阅读能力不强，没有软工的思想(我在说我自己)，所以读体量较小的工具我干脆把自己阅读的流程尽可能的表述出来，所以可能会有点啰嗦且没有重点QAQ
- 我读的是1.0.6版本的源代码，1.4版本增加了强制泛解析功能，如果有需要的话可以v我去读一遍。

0x02 文件结构
--
 - dict (字典目录，其中有dns服务器字典和子域名字典)
 - lib (里面就一个文件，主要是获取命令行宽度的，展示起来好看。) 
 - subdomain-brute.py(因为名字太长使用不方便我直接改成run.py了,主要逻辑都在里面所以只看这一个文件就行)

0x03 subdomain-brute.py
--

主文件我把它分为几个部分，先从开头开始看吧
### 程序入口
```python
if __name__ == '__main__':
    parser = optparse.OptionParser('usage: %prog [options] target.com')
    parser.add_option('-t', '--threads', dest='threads_num',
              default=60, type='int',
              help='Number of threads. default = 60')
    parser.add_option('-f', '--file', dest='names_file', default='dict/subnames.txt',
              type='string', help='Dict file used to brute sub names')
    parser.add_option('-i', '--ignore-intranet', dest='i', default=False, action='store_true',
              help='Ignore domains pointed to private IPs')
    parser.add_option('-o', '--output', dest='output', default=None,
              type='string', help='Output file name. default is {target}.txt')

    (options, args) = parser.parse_args()
    if len(args) < 1:
        parser.print_help()
        sys.exit(0)

    d = DNSBrute(target=args[0], names_file=options.names_file,
                 ignore_intranet=options.i,
                 threads_num=options.threads_num,
                 output=options.output)
    d.run()
    while threading.activeCount() > 1:
        time.sleep(0.1)
```
可以看到一共有4个参数：
- -t    线程数
- -f    字典文件
- -i    指定是否跳过内网IP
- -o    输出结果到指定文件 
一般使用不需要指定这些参数，继续往下读，实例化了DNSBrute类，传入了target和刚才传的一些参数。
实例化之后调用了run方法，最后用while阻塞进程直到线程结束

### DNSBrute类
```python
class DNSBrute:
    def __init__(self, target, names_file, ignore_intranet, threads_num, output):
        self.target = target.strip()
        self.names_file = names_file
        self.ignore_intranet = ignore_intranet
        self.thread_count = self.threads_num = threads_num
        self.scan_count = self.found_count = 0
        self.lock = threading.Lock()
        self.console_width = getTerminalSize()[0] - 2    # Cal terminal width when starts up
        self.resolvers = [dns.resolver.Resolver() for _ in range(threads_num)]
        self._load_dns_servers()
        self._load_sub_names()
        self._load_next_sub()
        outfile = target + '.txt' if not output else output
        self.outfile = open(outfile, 'w')   # won't close manually
        self.ip_dict = {}
        self.STOP_ME = False
```
处理了一些传参和初始化字典的东西
其中比较重要的是
```python
self.resolvers = [dns.resolver.Resolver() for _ in range(threads_num)]
self._load_dns_servers()
self._load_sub_names()
self._load_next_sub()
```
resolvers 把dns.resolver.Resolver()实例化后的对象传入了一个列表
\_load_dns_servers
- self.dns_servers = dns_servers		#dnsserver的列表
- self.dns_count = len(dns_servers)		#dnsserver列表的长度

\_load_sub_names
- self.queue	#queue对象，不理解消息队列也没关系，咱权当它是个list对象，里面装的是域名的字典

\_load_next_sub
- self.next_subs	#列表，二级子域的字典

### run方法/scan函数
```python
    def run(self):
        self.start_time = time.time()
        for i in range(self.threads_num):
            t = threading.Thread(target=self._scan, name=str(i))
            t.setDaemon(True)
            t.start()
        while self.thread_count > 1:
            try:
                time.sleep(1.0)
            except KeyboardInterrupt,e:
                msg = '[WARNING] User aborted, wait all slave threads to exit...'
                sys.stdout.write('\r' + msg + ' ' * (self.console_width- len(msg)) + '\n\r')
                sys.stdout.flush()
                self.STOP_ME = True
```
看到线程调用了self._scan，线程name为thread_num

```python
    def _scan(self):
        thread_id = int( threading.currentThread().getName() )
        self.resolvers[thread_id].nameservers.insert(0, self.dns_servers[thread_id % self.dns_count])
        self.resolvers[thread_id].lifetime = self.resolvers[thread_id].timeout = 10.0
        while self.queue.qsize() > 0 and not self.STOP_ME and self.found_count < 40000:    # limit max found records to 40000
            sub = self.queue.get(timeout=1.0)
            for _ in range(3):
                try:
                    cur_sub_domain = sub + '.' + self.target
                    answers = d.resolvers[thread_id].query(cur_sub_domain)
                    is_wildcard_record = False
                    if answers:
                        for answer in answers:
                            self.lock.acquire()
                            if answer.address not in self.ip_dict:
                                self.ip_dict[answer.address] = 1
                            else:
                                self.ip_dict[answer.address] += 1
                                if self.ip_dict[answer.address] > 2:    # a wildcard DNS record
                                    is_wildcard_record = True
                            self.lock.release()
                        if is_wildcard_record:
                            self._update_scan_count()
                            self._print_progress()
                            continue
                        ips = ', '.join([answer.address for answer in answers])
                        if (not self.ignore_intranet) or (not DNSBrute.is_intranet(answers[0].address)):
                            self.lock.acquire()
                            self.found_count += 1
                            msg = cur_sub_domain.ljust(30) + ips
                            sys.stdout.write('\r' + msg + ' ' * (self.console_width- len(msg)) + '\n\r')
                            sys.stdout.flush()
                            self.outfile.write(cur_sub_domain.ljust(30) + '\t' + ips + '\n')
                            self.lock.release()
                            try:
                                d.resolvers[thread_id].query('*.' + cur_sub_domain)
                            except:
                                for i in self.next_subs:
                                    self.queue.put(i + '.' + sub)
                        break
                except dns.resolver.NoNameservers, e:
                    break
                except Exception, e:
                    pass
            self._update_scan_count()
            self._print_progress()
        self._print_progress()
        self.lock.acquire()
        self.thread_count -= 1
        self.lock.release()
```
这么多代码属实看的我头疼，硬盘它！
前两行设置了dns.resolvers的nameservers和timeout
然后while循环取self.queue，每个名称尝试3次
用dns.resolvers.query请求dns服务器
底下就是处理应答，里面嵌套二级子域的爆破

0x03 参考文档
--
-  https://www.cnblogs.com/biglittleant/p/12955623.html
-  https://www.cnblogs.com/leozhanggg/p/10316974.html
-  https://github.com/lijiejie/subDomainsBrute
