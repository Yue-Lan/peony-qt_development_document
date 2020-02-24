# 基于GVFS扩展机制的搜索框架

如何去进行搜索，想必大家都有一个概念，但是如何在文件管理器中实现搜索，并且将结果展示出来，这就是一个比较棘手的问题了。

我研究过peony的搜索框架，旧版peony在gvfs文件系统上又封装了一层自己的vfs文件系统，使用uri-scheme来辨别它们（peony自己的搜索uri-scheme为x-peony-search:///），我想这对于新版peony的框架来说并没有可取之处，至少是不符合我的设计理念；为此我也调研过其它文件管理器的搜索框架，最让我受启发的就是libfm-qt的搜索框架——它并与libfm-qt强绑定，是glib层的框架。这样做对于自身框架的设计是极好的，因为这意味着不必在自己的框架上维护虚拟文件系统。

glib提供了gvfs scheme的扩展机制，它允许开发者在自己的应用中构建自己的文件系统（不能像file:///一样在所有的进程中使用），它们通过scheme进行区分，通过gfile接口进行统一的操作。只要实现了自己的vfs扩展，就可以使用现有的基于glib/gio/gvfs的框架，而不必修改已实现的代码。

要实现一个真正可用的vfs框架，我们需要做到2点——注册和实现GFile的接口。

对于文件搜索，我们把想要搜索的信息通过自定义的规则翻译成一段uri，通过这段uri获取的句柄会得到我们自定义的file接口实例，我们就能通过自己GFile接口实现的处理方法进行特殊的处理了。对于搜索来说，我们希望使用Peony-Qt的FileEnumerator进行遍历，那么至少要重新实现file的enumerate\_children接口，以及get到的enumerator的next\_file和next\_file\_async接口，当然为了与libpeony-qt的框架相匹配，还要实现其它的一些必要接口，这样我们就能够在现有框架中使用自定义的uri进行目录的遍历了。

libpeony-qt的文件搜索框架的实现可以参考源码

> [https://github.com/explorer-cs/peony-qt/tree/master/libpeony-qt/vfs](https://github.com/explorer-cs/peony-qt/tree/master/libpeony-qt/vfs)

搜索的算法目前采用的是常规的广度优先算法，为了节约时间，我使用了qt提供的容器和libpeony-qt的一些api，这些和在glib的框架下进行开发看起来可能就有些不伦不类了，但是从结果上看还是可以接受的，毕竟我只是为了针对新版peony文件搜索功能的需求。

