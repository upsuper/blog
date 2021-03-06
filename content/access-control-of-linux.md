Title: 探秘 Linux 权限控制
Date: 2009-07-06 16:08
Tags: Linux, Security
Category: Study
Slug: access-control-of-linux
Author: Xidorn Quan

众所周知，Linux 的权限控制虽然很简单，但却十分严格和有效的。（和 Windows 复杂却没用的权限控制形成鲜明对比……）由于最近编写测评机，希望利用 Linux 的高安全性做运行级恶意代码防护，因此就顺带地研究了一下 Linux 的权限控制。经过这次探秘，我对 Linux 的权限有了更新的认识，确实是一个很强大的东西啊！

由于本人的能力有限，文章中的不足和谬误也请大家多多指教！

我想，稍微接触过一段时间 Linux 的人都会对 Linux 的权限有些许了解，其中最重要的莫过于——很多命令需要加 sudo 才能运行，而且我们也知道，sudo 几乎无所不能——不能删的就 sudo rm、不能复制 sudo cp、不能移动 sudo mv……（目前我仅发现在部分虚拟文件系统中 sudo 也没有权限做这些事情……）那么，sudo 究竟是何方神圣，Linux 又是如何确定这些权限的呢？

说到 Linux 下的权限，一定要先说的是 Linux 下文件权限的控制。在 Linux 下，每个文件都有“所有者”和“所在组”这两个基本属性，而各种权限也是根据所有者、所在组和其他人划分的。每个文件的权限，最简单的情况下可以表示为一个3位八进制数，每一位八进制数表示一系列人的权限，如八进制数751就标示所有者有7的权限，所在组的其他人有5的权限，而既不是所有者也不在所在组的人只有1的权限。至于一位的八进制数表示的意义，我们应该将其进一步转换为3位二进制数，如7对应111，5对应101，1对应001。在这个二进制数上最高位如果为1则表示有读权限，第二位表示写权限，而最后一位表示执行权限。（说到这里我就想再插一句了，Linux 的文件是什么类型或可不可以执行，几乎完全不是根据扩展名，只有有执行权限的文件才能执行，而文件类型也是根据 MIME Type 来决定的。）

由于 Linux 里面大量的东西都可以转化为文件操作，因此这一简洁明了的设计解决了大多数权限控制的问题。不过文件归文件，那文件夹呢？文件夹应该说也是一类特殊的文件，因此也有权限控制。可对于文件夹，什么样叫“可读”，什么样叫“可写”？最奇怪的是，“可执行”？对于文件夹来说，可读就是可以列举文件列表，也就是 Linux 下的 ls 命令可以列出东西；可写就是可以在文件夹中创建文件（或许有人会问，为什么不是文件夹中文件可写？想一想~）；可执行是比较奇怪的……就是将这个文件夹当作当前文件夹的权限，在 Linux 命令中表现为 cd 是否可以使用。

好了，说完文件们，再来看看进程们。

进程的权限控制就更简单了，说白了就一句话：一个进程不能控制与其不是同一个用户下运行的进程，除非被控制的是它的子进程。事实上，一个进程有至少两个 UID 和 GID，它们分别是 EUID、RUID 和 EGID、RGID，其中，E 表示 effective，即所有权限控制参考的是 EUID 和 EGID，进程创建的文件的所有者和所在组也是 EUID 和 EGID 表示的用户和组。那么有人就问，R- 的那两个又是拿来干什么的呢？打酱油？非也，R 表示 real，即运行者的信息。其实一个进程可以改变自己这些 UID 和 GID，而 E- 可以修改为的值为其本身和对应的 R- 的值。不过一个程序在加载的时候 E- 和 R- 的值似乎都根据运行者的信息设置了相同的值，看起来好像没什么用，是吗？我们这里暂且不管他。

下面我们深入一步，看看进程调度中的 nice 值和进程的资源限制。不知道大家有没有用系统监视器调整过进程的 nice 值，nice 值是 Linux 核心调度进程的一个参考值，nice 值越高表示这个进程越不重要，优先级越低，越可以慢慢来；nice 值越低表示这个进程越重要，越要快些做。调过 nice 的人就知道，将可以控制的某个进程的 nice 值调高（优先级降低）是随便做的，但要把一个进程的 nice 值调低（提高优先级）却要输入密码（进入 root）。做过一些 Linux 相关的编程的人也应该知道，调紧一个可控进程的资源限制是可以随意调的，但调宽松也要由 root 来进行。

啊，root 出现了！root 是什么？root 对于 Linux 来说就是神。为什么这么说？你上面看到的所有权限控制，在 root 面前都是没用的，root 在 Linux 里可以为所欲为。是的，root 是一个完全不受权限限制的用户，这就是它可怕的地方。比如，我们在前面看到进程只能将 EUID 调整为 EUID 或 RUID，但如果 EUID 为 0（即为 root），这个进程将可以把 EUID 和 RUID 调整为任意值；同样的，这个进程将可以把 EGID 和 RGID 调整为任意值。

我们知道 Linux 内核加载的时候开始执行的进程 init 是以 root 运行的，而后 init 加载各个启动脚本，把各项服务封入单独的用户运作，最后根据登入信息加载用户进程。这整个过程中，只有由上而下，由 root 到普通用户的过程。理由就是 root 可以任意调整 E- 和 R-。不过如果仅是这样，这个系统估计什么也干不了……

我在原来的一篇日志中提到过 Linux 创建子进程和运行程序的方法——fork 和 exec* 函数：fork 运行成功返回两次，一次在生成的子进程中返回0，另一次在调用的父进程中返回子进程的 pid。此时，子进程迅速继承了父进程的所有权限、变量等等等等。（Linux 的子进程采用 copy-on-write 技术分享父进程的内存）而我们前面又知道，一个进程只能让自己从 root 到普通用户，优先级从高到低，资源限制从宽到严，就像水只能从高流向低一样。但，人是往高出走的，有时我们需要 root 的权限，问：路在何方？

就在这时候，sudo 出现了！sudo 在 Linux 中可为无人不知无人不晓。对于 sudo 来说，只要你这个用户在 sudoer 的列表中，输入你的密码就可以让你成为 root 了，或者你输入其他用户的密码可以让你使用其他用户。可是，我们前面刚刚说过，进程只能让自己从 root 变成普通用户，那这 sudo 又是哪冒出来的呢？

这又要回到文件权限设置了。其实，文件权限实际上有4位八进制数。我们原来说的是3位，那还有一位是什么呢？那一位对应的3个二进制位又是三个开关，分别标示 SUID、SGID 和粘着位。SUID 是什么？它表示 saved set-user-ID，设置了它的程序在运行时有了新的选择——程序所有者的权限！也就是说，EUID 的取值在 EUID、RUID 之外多了一个选择——SUID。事实上，有设置 SUID 的程序在执行时都会自动将 EUID 设置为 SUID，这时 EUID 和 RUID 就不再是同一个值了，我们前面又知道，只要 EUID 为0这个程序就无敌了。这个过程在 fork 中无法完成，我猜想是在 exec* 函数的调用过程中进行的。我们看到，sudo 这个程序的所有者是 root，而又有设置 SUID，这表示，我们一运行 sudo，它就以 root 在运行了！很神奇不是么？通过 SUID 这种特别的机制，原来许多不可能的事成为了可能。SGID 表示的含义亦是类似的；而至于粘着位表示这个文件对于任何人都可写，但只有所有者能删除。

到这里，我把 Linux 基本的权限控制机制都说了一番。当然，为了更精确地进行权限控制，Linux 内核还引入了 Linux 安全模块，并可以加入 SELinux 和 AppArmor 等增强的安全机制，这让 Linux 更加安全和坚不可摧。（Windows 的权限控制在这些面前简直就是幼稚园小孩在大学教授面前……）这些我就不细致展开了，更多的信息网上很容易找到。

参考资料：

* [权限管理 – 开源世界旅行手册](http://linuxtoy.org/docs/guide/ch17s08.html)
* [Solaris下究竟如何使用setuid/seteuid/setreuid](http://www.cndw.com/tech/server/2006040430540.asp)
* [the Linux Cross Reference](http://lxr.linux.no/)
* [Linux系统进程的几个用户ID及其转换方法](http://tech.ccidnet.com/art/741/20090623/1806969_1.html)
* [用户信息 /etc/passwd，getuid(), getpwuid()](http://idcnews.net/html/edu/20070101/291393.html)
* [C语言系统资源控制（getrlimit && setrlimit)](http://hi.baidu.com/phps/blog/item/7e3ba44410cf9580b3b7dc81.html)
* [permission problem with os.setuid](http://bytes.com/groups/python/36126-permission-problem-os-setuid)
