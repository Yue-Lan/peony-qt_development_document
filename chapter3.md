# peony-qt与model/view编程

## 什么是model/view编程？

model/view编程是一种前后端分离的编程模式，可以看做是设计模式的一种，相信大家对MVC都略有耳闻，qt的model/view编程就是与其类似的一种东西。与MVC不同的是，qt的model/view编程已经提供好了一组可以复用的api来实现这种架构，我们只需要实现既定的接口就可以了。

## model/view编程的思想

假设有这样一个情况，我们有若干组相同结构的数据，它们由一定的规则存储在应用程序中，现在我们需要把它们按照既定的顺序显示出来，那么我们该怎么做呢？

最直接的想法——既然每个数据都一样，那么我们可以构造一个统一的widget用于显示单个数据，然后通过layout将数据摆放出来。但是这样做可能会遇到诸多麻烦，比如如果我们想要修改视图的布局，或者是数据是以树状结构存储的，这个时候要做的工作就变得不明确了。对于文件管理器，它对显示文件的细节内容和显示文件的方式都提供了不少于两种的解决方案，这时如果采用以上的思路进行，那么看起来会是一项十分艰巨的任务。

事实上，在peony中，文件管理器实现的思路主要还是偏向于这种方案，这也导致了整个peony的代码复杂度变得十分的大，在10多年前，这些框架都还不发达的甚至没有诞生的时候，所有的思路都要从头构建，这也导致了一个文件管理器的开发者不得不完美的掌握前端和后端以及二者交互的技术，其实我还是挺佩服那个时候的人的，换成是我可能就直接放弃了。那个时代人人喊着MVC，要做到前后端分离，但是不论是MVC还是直接裸写，实现一个文件管理器这样的大型项目都绝对不是一个简单的工作。

直到现在，哪怕gtk也已经推出了类似于，model/view编程的api，Gtk阵营的文件管理器依然难以进行这方面的迁移——前人写的东西确实很成熟，但确实与现代编程的思路和体系差距太大了。时代是在进步的，抱着过去的文明产物不放很难迎接新时代的到来，也许若干年后现在的智慧也会被淘汰，但至少我希望现阶段能够不断推陈出新。哪怕只是摸到前沿技术的背影，对于开发而言也都是有非凡意义的。

回归正题，自qt4引入了model/view编程之后，之前说道的问题总算是有了一个比较靠谱的解决方案，我们通过一个例子来理解它蕴含的理念。

### 使用已有的model

我们首先创建一个Qt的Gui application，在main文件中编写如下代码：

```c
#include <QApplication>
#include <QFileSystemModel>
#include <QSplitter>
#include <QTreeView>
#include <QListView>
#include <QDir>

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);
    QSplitter *splitter = new QSplitter;

    QFileSystemModel *model = new QFileSystemModel;
    model->setRootPath(QDir::currentPath());
    QTreeView *tree = new QTreeView(splitter);
    tree->setModel(model);
    tree->setRootIndex(model->index(QDir::currentPath()));

    QListView *list = new QListView(splitter);
    list->setViewMode(QListView::IconMode);
    list->setModel(model);
    list->setRootIndex(model->index(QDir::currentPath()));
    splitter->setWindowTitle("Two views onto the same file system model");
    splitter->show();
    return app.exec();
}
      

```

执行这段代码，大概会生成如下效果

图片。。。。。。。。。。。。

这段代码其实是从qt官方文档中截取的一段示例代码，稍作修改得出的，我们可以看出，通过model和view，我们能够使用不同的view展示同一个model。实际上，只要满足接口，model和view是可以即插即用的。

其实model/view编程的思想并不局限于此，但对于文件管理器来说这就是最主要的一点了——不论上层UI如何变动，下层始终只要维护一个model就够了（或者根据应用情景不同对于已有的model进行适当的修改）。这种编程模式非常符合目前文件管理器的设计思路。

以上代码中的api都是由Qt提供的，我们也可以看出Qt给出的文件系统model十分的强大，然而它并非没有缺陷。最明显的一点，QFileSystemModel不具备读取本机文件系统之外的文件的能力，这使得它的应用场景变得十分狭小。比如，如果我想把一个文件放进回收站，或者从回收站还原一个文件，使用QFileSystemModel就很难了。

GVfs提供了一些非常遍历的特殊uri作用域，其中就包括我之前所说的回收站，实际上Peony也是基于GVfs的，那么我们的Peony-Qt也应该对这一部分进行重构和迁移才行——我们需要实现类似于QFileSystemModel并且具有GVfs特性的model，这个目标非常明确，但是具体该如何去做呢？

## 实现自己的model

我们大概知道了model/view编程的思路，model和view是有一个抽象框架的，我们利用这个框架可以实现数据的model化，具体是怎么做的呢？我们需要一步一步的分析。

我建议在阅读本章之前，你最好能够了解gio的一些知识，比如GFileEnumerator的使用等，我之前写过一本gitbook，专门有一章介绍了gio常用接口的简单用法，有了这些知识，才能够完整的理解接下来我要阐述的思路和内容 。

给上之前的文章的链接——[https://github.com/Yue-Lan/gitbook-fm/tree/master/gvfsgio](https://github.com/Yue-Lan/gitbook-fm/tree/master/gvfsgio)

#### 在有model之前，还是得有数据

巧妇难为无米之炊，空有model的框架没有数据肯定是不行的，我们可以把整个文件系统的结构看做是以uri域为根节点的多叉树，每一个节点都储存了文件本身的信息。

在数据这方面，我们需要关注如何能够获取期望的数据，并且保持一个相对高效的获取过程。

如果我们以一个文件夹作为跟节点，最终如果要显示该目录下的所有文件，我们首先需要遍历这个节点的所有子节点，对于每个遍历到的子节点，我们要检索他们的信息，如果我们的视图是树状的，我们的子节点也应该能够和现在的根节点一样，能够获取和保存它的子节点并且遍历信息，整个数据结构是典型的递归模式。

我将以上这些要素抽象成了一些class，遍历文件的FileEnumerator，查询信息的FileInfoJob，存储信息本身也可以代表文件的FileInfo，在此之上，构建了树状结构的FileItem。

我们也许不会在model创建的一开始就获取所有的文件节点，这样太低效了，那么如何动态的获取和释放子节点的资源也是一个值得仔细考量的点。在Peony中，文件管理器的目录是实时监听的，我们在目录遍历完成之后也应该能够对目录的变化做出回应。

针对上面第一个问题，我将FileItem设计成可以展开和收缩的模式，针对第二个问题，我增加了FileWatcher类用于监听和通知的实现。

通过这些类，应该完整的底层gvfs数据层就基本上实现了，只是我们现在暂时还没什么办法有让它们显示在视图上，剩下来的照理来说就是model的工作了。

#### model与数据的关系

在讲述model接口实现的方法之前，我提出几个model的关键设计理念，我们需要在这些理念的指导下才能正确的实现我们自己的model。下面我们来罗列一下：

* model和数据的关系通过QModelIndex表现出来，每一个有效的index实际上都代表着一个数据元素，在这里也就是FileItem，每一个index都有一个指向该元素的internal pointer，可以使得index向元素实例进行类型转换。这样的设计使得元素和model被绑定在一起，如果一个FileItem脱离了model，那么它也不会有对应的index，所以不能算一个完整的元素。
* index本身是树状的结构，每一个index都有一个parent index，多个index可以指向同一个parent。parent可能是非法的，如果parent非法，通常代表这个元素位于view的顶层，对FileItem而言，则表示为是根节点下的直接子节点，我们的icon view显示一个文件目录时，该目录本身不会被显示出来，这个道理是一样的。
* 每一个index都有一个rowCount和columnCount，如果他们都不为0，则说明该节点是有子节点的，为了正确的显示文件系统，我们需要对每个item对应的index的rowCount和columnCount做出正确的判断，在model中，判断index的rowCount和columnCount的行为是自动并且根据parent递归触发的，所以我们只需要关注当前index的count即可。
* model会根据parent index的rowCount和columnCount来生成下一个index，index的生成通常通过createIndex\(row, column, data\)这种形式生成。

#### model的选型与接口的实现

Qt提供的model有很多，有的是纯虚的model，有的是纯实的model，有的是半虚的model，model的虚实程度是根据model已经实现的接口数量来决定的，像QAbstractItemModel就是一个纯虚的model，一个接口都没有实现，而QFileSystemModel则实现了QAbstractItemModel的全部接口，所以是纯实的model，这种model没有太多改动的空间了。介于其中的还有QListModel，QTableModel，QStantardItemModel等，它们或多或少的实现了部分接口，并且做了相应的封装，这些model将会适用于不同的场景。

一般情况下我们不会优先考虑基于QAbstractItemModel实现我们自己的model，然而在这里为了实现GVfs model苛刻的要求，我还是迫不得已的选择了QAbstractItemModel作为实现model的接口。

我们首先看一看一个QAbstractItemModel究竟有哪些接口：

如果你使用QtCreator开发，那么你能够找到相关model的实现模板，我们在新建一个文件或项目时，选择Qt-&gt;item model。



图片。。。。。。。。。。

创建时有很多选项，总之我们都勾上。

图片。。。。。。。。。。



最后得到的代码如下：

```c
#ifndef TESTMODEL_H
#define TESTMODEL_H

#include <QAbstractItemModel>

class TestModel : public QAbstractItemModel
{
    Q_OBJECT

public:
    explicit TestModel(QObject *parent = nullptr);

    // Header:
    QVariant headerData(int section, Qt::Orientation orientation, int role = Qt::DisplayRole) const override;

    bool setHeaderData(int section, Qt::Orientation orientation, const QVariant &value, int role = Qt::EditRole) override;

    // Basic functionality:
    QModelIndex index(int row, int column,
                      const QModelIndex &parent = QModelIndex()) const override;
    QModelIndex parent(const QModelIndex &index) const override;

    int rowCount(const QModelIndex &parent = QModelIndex()) const override;
    int columnCount(const QModelIndex &parent = QModelIndex()) const override;

    // Fetch data dynamically:
    bool hasChildren(const QModelIndex &parent = QModelIndex()) const override;

    bool canFetchMore(const QModelIndex &parent) const override;
    void fetchMore(const QModelIndex &parent) override;

    QVariant data(const QModelIndex &index, int role = Qt::DisplayRole) const override;

    // Editable:
    bool setData(const QModelIndex &index, const QVariant &value,
                 int role = Qt::EditRole) override;

    Qt::ItemFlags flags(const QModelIndex& index) const override;

    // Add data:
    bool insertRows(int row, int count, const QModelIndex &parent = QModelIndex()) override;
    bool insertColumns(int column, int count, const QModelIndex &parent = QModelIndex()) override;

    // Remove data:
    bool removeRows(int row, int count, const QModelIndex &parent = QModelIndex()) override;
    bool removeColumns(int column, int count, const QModelIndex &parent = QModelIndex()) override;

private:
};

#endif // TESTMODEL_H
```

```c
#include "testmodel.h"

TestModel::TestModel(QObject *parent)
    : QAbstractItemModel(parent)
{
}

QVariant TestModel::headerData(int section, Qt::Orientation orientation, int role) const
{
    // FIXME: Implement me!
}

bool TestModel::setHeaderData(int section, Qt::Orientation orientation, const QVariant &value, int role)
{
    if (value != headerData(section, orientation, role)) {
        // FIXME: Implement me!
        emit headerDataChanged(orientation, section, section);
        return true;
    }
    return false;
}

QModelIndex TestModel::index(int row, int column, const QModelIndex &parent) const
{
    // FIXME: Implement me!
}

QModelIndex TestModel::parent(const QModelIndex &index) const
{
    // FIXME: Implement me!
}

int TestModel::rowCount(const QModelIndex &parent) const
{
    if (!parent.isValid())
        return 0;

    // FIXME: Implement me!
}

int TestModel::columnCount(const QModelIndex &parent) const
{
    if (!parent.isValid())
        return 0;

    // FIXME: Implement me!
}

bool TestModel::hasChildren(const QModelIndex &parent) const
{
    // FIXME: Implement me!
}

bool TestModel::canFetchMore(const QModelIndex &parent) const
{
    // FIXME: Implement me!
    return false;
}

void TestModel::fetchMore(const QModelIndex &parent)
{
    // FIXME: Implement me!
}

QVariant TestModel::data(const QModelIndex &index, int role) const
{
    if (!index.isValid())
        return QVariant();

    // FIXME: Implement me!
    return QVariant();
}

bool TestModel::setData(const QModelIndex &index, const QVariant &value, int role)
{
    if (data(index, role) != value) {
        // FIXME: Implement me!
        emit dataChanged(index, index, QVector<int>() << role);
        return true;
    }
    return false;
}

Qt::ItemFlags TestModel::flags(const QModelIndex &index) const
{
    if (!index.isValid())
        return Qt::NoItemFlags;

    return Qt::ItemIsEditable; // FIXME: Implement me!
}

bool TestModel::insertRows(int row, int count, const QModelIndex &parent)
{
    beginInsertRows(parent, row, row + count - 1);
    // FIXME: Implement me!
    endInsertRows();
}

bool TestModel::insertColumns(int column, int count, const QModelIndex &parent)
{
    beginInsertColumns(parent, column, column + count - 1);
    // FIXME: Implement me!
    endInsertColumns();
}

bool TestModel::removeRows(int row, int count, const QModelIndex &parent)
{
    beginRemoveRows(parent, row, row + count - 1);
    // FIXME: Implement me!
    endRemoveRows();
}

bool TestModel::removeColumns(int column, int count, const QModelIndex &parent)
{
    beginRemoveColumns(parent, column, column + count - 1);
    // FIXME: Implement me!
    endRemoveColumns();
}

```

这就是model的模板了，我们需要基于这个模板重写各个方法的内容，由于我们的数据实际上是存储在FileItem中的，所以FileItem和model的联系会成为一个难点，我在这里给出我在实现FileItemModel时的思路。

