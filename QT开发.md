# 安装教程和学习教程

本文是我自己在学习QT的时候的心得，基本依照以下博客或者教程编写，方便开发。

> [(4条消息) qt6.4.0+visual studio2022+opencv配置教程（2022年最新版）_~幻化成风~的博客-CSDN博客_qt最新版本](https://blog.csdn.net/memorywithyou/article/details/126607163?spm=1001.2014.3001.5506)

> [【北京迅为】嵌入式学习之QT学习篇](https://www.bilibili.com/video/BV1tp4y1i7EJ?p=7&vd_source=b861809de5579169c2e8682cde41b2cd)

> [收藏】QT图形框架编程开发（层层到肉）_C++图形用户界面开发框架](https://www.bilibili.com/video/BV1Wf4y1Y7uh/?p=3&share_source=copy_web&vd_source=b861809de5579169c2e8682cde41b2cd)

***
# 比较好的框架

[Linloir/GraphBuilder: A visualized tool to create a graph (github.com)](https://github.com/Linloir/GraphBuilder)

做成上位机需要进行一些剪裁。

# 开发中遇到一些问题

1. **遇到警告：**QT警告**Slots named on_foo_bar are error prone**

    **原因：**

    这个警告的出现，是因为我们在处理信号–槽关系时，是通过 ui designer中的"Go to slot" ，让程序自动生成。
    而这种自动生成的弱点就是也许有一天，你在 ui designer中改了控件的名字，但此时编译也不会报错。程序还是正常跑，编译也不提示错误。
    这样，控件就相当于连不到槽函数上去了，就失效了。

    **解决：**

    - 不要通过ui designer的 “Go to slot” 自动生成 信号–槽的连接关系。手动建立该关系即可。例如：
        connect(toolButton, &QPushButton::clicked,
        this, &YourClassName::nameOfYourSlot);
    - 不要管这个警告，无视他就行了。因为这只是一个善意的提醒。

![image-20230112152452273](./QT开发.assets/image-20230112152452273.png)

2. **问题：**在开发Crucis上位机的时候，QPlainTextEdit控件显示的效果与正点原子差距较大，以下为两者比较图

    ![image-20230112152850550](./QT开发.assets/image-20230112152850550.png)

    <center><p>自制的上位机</p></center>

![image-20230112152906592](./QT开发.assets/image-20230112152906592.png)

<center><p>正点原子的XCOM</p></center>

从中会发现两个问题，第一是换行，源代码都是\r\n，但是自制上位机明显多换了一行；第二是自制上位机出现了串行的问题，J 2的数据到了下一行，这样会使得后面的数据库无法分析和存储数据。

**解决：**我分析是QT中接收槽函数出现了问题，最后发现是readAll( )和appendPlainText( )两个函数的问题。我分别换成了readLine( )和insertPlainText( )才解决问题。

**具体分析：**

- 下位机发送都是一行行发，发送速度较快，readAll( )会使得把后面的一些数据都读走了，出现了串行问题，要限制一下读取范围。readLine( )就是读到第一个\n结束

- appendPlainText( )的函数功能是Appends a new paragraph with text to the end of the text edit.就是新建一个段落，所以出现了换行再换行的问题。而insertPlainText( )的函数功能是Convenience slot that inserts text at the current cursor position.就是在当前光标位置插入文本，效果就和正点原子的类似。

```c++
 QString buf;
    buf = QString(serialPort->readAll()); //应该换成readLine( )
    ui->ReceivePTE->appendPlainText(buf); //应该换成insertPlainText(buf)
```

3. **问题：**上位机在接收STM32数据的时候，传输速率和数据量还没到达最大，就已经出现了卡顿情况

    **解决：**

    具体要求是STM32上传数据到上位机，上位机需要分析头部数据，并且存入数据库，最后显示在Table View中

    ```c++
    /* 老程序，速度慢 */
    //如果是JY901的数据
    if(data.at(0) == 'J')
    {
     	//查询数据表中是否已经存在数据
        QString checkstr = QString("SELECT * FROM jy901 WHERE ID = '%1'").arg(data.at(1));
        QSqlQuery query(checkstr);
        if(query.next()) //存在数据，执行更新操作
        {
            QString updatestr = QString("UPDATE jy901 SET Data = '%2', X = %3, Y = %4, Z = %5 WHERE ID = %1")
                                .arg(data.at(1),data.at(2),data.at(3),data.at(4),data.at(5));
            query.exec(updatestr);
        }
        else
        {
            QString insertstr = QString("INSERT INTO jy901 VALUES(%1,'%2',%3,%4,%5)")
                                .arg(data.at(1),data.at(2),data.at(3),data.at(4),data.at(5));
            query.exec(insertstr);
        }
        //Table View显示函数
    }
    ```

    分析上述原始程序，每次读取到新数据就打开一次数据库并且写入（更新），同时还要显示在Table View中，这样十分耗时。例如，JY901的四组数据，每组数据来都需要打开关闭数据库4次，严重拖慢了整体速度。下面是对此程序的优化。

    ```c++
    if(data.at(0) == 'J')
    {
        //查询数据表中是否已经存在数据
        QString checkstr = QString("SELECT * FROM jy901 WHERE ID = '%1'").arg(data.at(1));
        QSqlQuery query(checkstr);
        if(query.next() == false)
        {
            QString insertstr = QString("INSERT INTO jy901 VALUES(%1,'%2',%3,%4,%5)")
                                .arg(data.at(1),data.at(2),data.at(3),data.at(4),data.at(5));
            query.exec(insertstr);
        }
        else
        {
            JYdatalist << data; //将数据存入容器中 这里的容器定义为QVector<QStringList> JYdatalist;
            JYnum+=1;
            if(JYnum == 4) //收集完数据
            {
                db.transaction();   //事务处理开始
                for(int i=0;i<4;i++)
                {
                    QString updatestr = QString("UPDATE jy901 SET Data = '%2', X = %3, Y = %4, Z = %5 WHERE ID = %1")
                               .arg(JYdatalist[i].at(1),JYdatalist[i].at(2),JYdatalist[i].at(3),JYdatalist[i].at(4),JYdatalist[i].at(5));
                    query.exec(updatestr);
                }
                db.commit();    //事务处理结束
                JYdatalist.clear();
                JYnum = 0;
    		   //Table View显示函数
         	}
    	}
    }
    ```

    经过优化过后的程序，主要使用了C++的STL容器和SQLite中的事务处理函数。将收集到的数据存入容器中，等收集完成后再开启事务一次性写入到数据库中。事务处理函数在处理大量数据时，有着显著的优势。

4. **问题：**在写多线程数据库访问时，出现控制台错误**requested database does not belong to the calling thread.**
   
    **原因：**创建数据库连接和使用数据库连接不在一个线程中
    
    **解决：**在子程序访问数据库时，需要指定数据库对象名。例如`QSqlQuery query(checkstr,db);`
    
    这里的db就是需要访问的数据库对象名。
    
5. 在使用第三方库时，上传到GitHub，有部分.dll动态库无法上传，例如SDL2.dll。在打包的时候，打包工具也不会将该动态库放入文件夹中，需要自己手动添加。

***

# VS和QTcreator之间的转化

- 可以实现从VS到QTcreator的导入，在项目处选择create .pro文件，同时取消勾选create .pri文件选项。然后返回项目文件夹，双击打开pro文件进入QTcreator，需要在pro文件中添加模块定义，例如QT+=core gui widget，即可编译运行。

- 也可以实现从QTcreator到VS的导入，将QT生成的pro文件路径(创建工程时使用qmake)。进入VS，在拓展处open QT project .pro。可能会存在中文编码问题，需要修改，可以参考视频

> [Qt 5.14.2 下载、安装、使用教程，Qt+vs2019开发环境搭建](https://www.bilibili.com/video/BV1r54y1G7m4?p=7&vd_source=b861809de5579169c2e8682cde41b2cd）

## Qmake与Cmake的区别
> [(4条消息) Qmake VS Cmake_vbskj的博客-CSDN博客_qmake和cmake区别](https://blog.csdn.net/vbskj/article/details/7792061)

***

# QT注意事项

1. QT使用qDebug打印

2. 如果是输入密码的话，需要将line edit中的`echomode`改成password

3. 在QT中main.cpp中的固定写法为：

    ```c++
    int main(int argc, char *argv[])
    {
        QApplication a(argc, argv);
        Widget w;
        w.setWindowTitle(QString("串口调试助手")); //该行用来更改窗口标题，可加可不加
        w.show();
        return a.exec();
    }
    ```
4.emit是Qt的关键字，标记当前是发射信号，例如 `emit mySignal();`

***
# ~~QT中文编码转换~~

- 在默认情况下，QT可以正确理解UFT-8的编码，将其自动转化为内部的Unicode编码，如果使用Windows中常用的GBK编码，将可能出现乱码问题。

- 通过QTextCodec实现编码转换

    ```c++
    Qtextcodec *codec = QTextCode::codeForName("GBK");
    QString string = code->toUnicode("GBK编码的中文字符串")
    ```

    > <font color=red>Tips:</font>此类在QT6中被移除，使用前需在pro文件中添加代码
    >
    > [QT6中QTextcodec头文件找不到 - AlexSun_2021 - 博客园 (cnblogs.com)](https://www.cnblogs.com/AlexSun-2021/p/16043500.html#:~:text=QT6中QTextcodec头文件找不到,选择QT5的兼容模块 然后再重新打开QT6，在你的项目代码上添加一句代码就可以了)

***

# 父窗口

父窗口的析构函数回自动销毁所有子窗口的对象，所以子窗口对象是通过new操作符创建的，可以不显式执行delete操作，不会造成内存泄漏。

## 常用的父窗口类

- QWidget
- WMainWindow （主窗口）
- QDialog（对话框）

## 设置窗口的位置和大小函数

`void move(int x,int y);`

`void resize(int x,int y);`

## 在父窗口上创建对象

举例在<font color="red">栈区</font>创建一个label

`Qlabel label("我是标签",&widget); //后续使用.操作符`

举例在<font color="red">堆区</font>创建一个label

`Qlabel* label = new Qlabel("我是标签",&widget); //后续使用->操作符`

***

# 信号、槽、关联

## 信号的定义

```c++
class XX:public QObject
{
    Q_OBJECT	//宏定义，源对象编译器，语法拓展
signals:
    void signal_func(); //信号函数
};
```

> <font color=red>Tips:</font>信号函数只需声明，不能写定义

## 槽的定义

```c++
class XX:public QObject
{
    Q_OBJECT	//宏定义，源对象编译器，语法拓展
public slots:
    void sslot_func(); //槽函数
};
```

> <font color=red>Tips:</font>槽函数可以连接到某个信号上，当信号被发射时，槽函数将被触发和执行，另外槽函数也可以当作普通成员函数调用

槽函数声明写在对应类的private slots下<font color =red>（目前VS中使用ui不能转到槽的操作）</font>

## 关联

- 自动关联	在ui界面右键选择转到槽，后面再生成的函数块中写入执行的代码

- 手动关联	使用connect函数 connect(A,SIGNAL(B),C,SLOT(D));    <font style = "background:green">当对象A发出信号B，执行对象C中的槽函数D</font>

    函数原型：

    `QMetaObject::Connection QObject::connect(const QObject *sender, const char *signal, const QObject *receiver, const char *method);`

    参数：

    - sender：信号发送对象指针
    
    - signal：要发送的信号函数，可以使用`SIGNAL`宏进行类型转换
    
    - receiver：信号接收对象的指针
    
    - method：接收信号后要执行的槽函数，可以使用`SLOT`宏进行类型转换
    

> <font color=red>Tips:</font>可以多对一或者一对多关联
>
> 前提是信号函数的参数个数要==大于或者等于==槽函数的参数个数（可以有缺省值）

***

# MainWindow添加菜单栏

**QMenuBar** --菜单栏类，即下图中红色区域标记，菜单栏类给窗口提供水平菜单栏，此菜单栏占用窗口上方区域，垂直高度不变，水平宽度为窗口宽度，可随窗口大小变化而变化。如下图中“测试”，“test1”，"test2"所在的栏几位QMenuBar

**QMenu** --菜单项，即下图中绿色区域，下图中“测试”,"test1","test2"都是一个独立的菜单，包含各个子菜单。**QMenu还可以用来创建弹出菜单**。

**QAction** --子菜单，即下图中蓝色区域标记的内容，一个子菜单对应一个操作。

![菜单栏](./QT开发.assets/菜单栏.png)

<center><p>菜单示例</p></center>

一般使用QT创造师进行ui编辑，进入“设计”页面，进入如下图所示的界面，具体操作方法见如下两张图，**注意：输入菜单名称后一定要按==“Enter”==键才能生效**。

![创造师](./QT开发.assets/创造师.png)

<center><p>ui设计界面操作</p></center>

![创造师2](./QT开发.assets/创造师2.png)

<center><p>修改相关属性</p></center>

> [Qt基础之菜单栏 - kyzc - 博客园 (cnblogs.com)](https://www.cnblogs.com/kyzc/p/11962903.html)

***
# 事件

- 在Qt中，事件被封装成对象，所有的事件对象类型都继承自抽象类QEvent
- 当事件发生时，首先被调用的是QObject类中的虚函数event( )，其参数(QEvent)标识了具体的事件类型
- 在Qt桌面应用开发中，QWidget类覆盖了其基类中的event( )虚函数，并根据具体事件调用具体事件处理函数：
    - `void QWidget::mousePressEvent(QMouseEvent* e);//鼠标按下事件`
    - `void QWidget::mouseReleaseEvent(QMouseEvent* e);//鼠标抬起事件`
    - `void QWidget::mouseMoveEvent(QMouseEvent* e);//鼠标移动事件`
    - `void QWidget::paintEvent(QPaintEvent* e);//绘图事件`

## 绘图事件
- 通过绘图事件，可以实现自定义的图像绘制，即QWidget类的paintEvent()虚函数被调用

- 如果希望在自己的窗口中显示某个图像，在Qwidget的窗口子类中重写绘图事件函数paintEvent，在其中可用QPainter实现指定的图像绘制、渲染等操作

- rect和painter的坐标系不一致，会有偏移，使用改语句使得坐标系统一，具体代码见==案例五==

    ```c++
    rect.translate(ui->frame->pos());
    ```

## 定时器事件

1.Qt通过两套机制为应用程序提供定时服务

   - 定时器事件，由QObject提供
   - 定时器信号，由QTimer提供

2.通过定时器事件实现定时器

   `int QObject::startTimer(int interval);`启动定时器，每隔x毫秒返回定时器ID

   `void QObject::timerEvent(QTimerEvent*)[virtual];`定时器事件处理函数

   `void Qobject::killTimer(int id);`关闭参数ID的定时器

## 鼠标和键盘事件

### 鼠标事件

QWidget类定义了以下的虚函数提供对鼠标事件的处理，其参数QMouseEvent描述了鼠标事件的细节，如引发事件的鼠标按键、鼠标所在位置等

- `virtual void mousePressEvent(QMouseEvent* e);//鼠标按下`
- `virtual void mouseReleaseEvent(QMouseEvent* e);//鼠标抬起`
- `virtual void mouseDoubleClickEvent(QMouseEvent* e);//鼠标双击`
- `virtual void mouseMoveEvent(QMouseEvent* e);//鼠标移动`

### 键盘事件

QWidget类定义了以下虚函数提供对键盘事件的处理，其参数QKeyEvent描述了键盘事件细节，如引发事件的键盘按键、文本等

- `virtual void keyPressEvent(QKeyEvent* e);//按键按下`
- `virtual void keyReleaseEvent(QKeyEvent* e);//按键抬起`

***

# 使用外部资源

## 静态加载

1.导入素材

- 在项目上右键 添加文件，选择Qt->Qt Resource File

- 打开.prc文件，右键open in editor，对前缀进行保存，一般是/，然后导入文件

    > <font color=red>Tips:</font>静态加载一般适用于资源较少或者资源名有规律的情况下使用，存在大量的资源最好还是使用下面的动态加载

2.引用素材

- 在label上右键，改变样式表
- 点击添加资源旁边的小箭头，加入合适的资源

![image-20221115132817263](QT开发.assets/image-20221115132817263.png)

<center><p>添加资源的示例</p></center>

## 动态加载

当文件的名字不是按照顺序来的时候，就需要使用动态加载了，可以不需要列举所使用的全部资源，而是在程序执行的时候去搜索。

- 主要是使用QDir类实现，具体实现方法可以看==案例六==中的`loadPhotos`函数，该函数实现了在当然目录寻找图片，同时递归在子目录寻找图片。

- 案例六中还使用了QVector来存储图片，因为图片是不定个数的，使用普通数组容易造成浪费。

- 语法是`QVector<类> 变量名` 该容器也可以使用数组的方式访问，例如变量名[i]。

***

# 布局

水平布局 垂直布局 栅格布局  

如果纯代码实现，可以使用`QHBoxLayout`类以及`QVBoxLayout`类，以下为示例，水平布局器从左往右，垂直布局器从上往下。

```c++
    //创建水平布局器
    QHBoxLayout* layout = new QHBoxLayout(this);
    //按水平方向加入布局器
    layout->addWidget(m_editX);
    layout->addWidget(m_label);
    layout->addWidget(m_editY);
    layout->addWidget(m_button);
    layout->addWidget(m_editZ);
    setLayout(layout);

	//创建垂直布局器
    QVBoxLayout* layout = new QVBoxLayout(this);
    layout->addWidget(m_label);
    layout->addWidget(m_button);
    setLayout(layout);
```



布局的意义在于窗口缩放不会改变对象的相对位置。

因此需要合理布局，在ui文件中布局后编译会自动产生对应的C++代码。

---

# 样式表

通过setStyleSheet方法设置样式表

需要在代码中先加入`xxxxxx->setView(new QListView()); `  //下拉列表样式表才能生效

下面是给QComboBox增加样式

```c++
    QString sheet = "\
            QComboBox {\
                border: 1px solid gray; \
                border-radius: 3px;\
                padding: 1px 18px 1px 3px; \
                color: #000;\
                font: normal normal 15px Microsoft YaHei;\
                background: transparent;\
            }\
            QComboBox QAbstractItemView {\
                font-family: Microsoft YaHei;\
                font-size: 15px;\
                border:1px solid rgba(0,0,0,10%);\
                border-radius: 3px;\
                padding:5px;\
                background-color: #FFFFFF;\
            }\
            ";

    PIDComboBox->setView(new QListView());
    PIDComboBox->setStyleSheet(sheet);
```

[QComboBox样式设置——Qt_qcombobox::drop-down_十年之少的博客-CSDN博客](https://blog.csdn.net/xiaopei_yan/article/details/107404698)

***

# 界面的切换

在项目右键，创建文件，选择Qt设计师界面类。在新的界面中进行布局，可以使用按钮进行两个界面之间的切换。

假如新创建的窗口名为ctrl

1.包含头文件ctrl.h

2.在转换按键槽函数中写入以下代码实现切换

```c++
void Widget::xxxx_Slots()
{
    ctrl *ct = new ctrl;
    ct->setGeometry(this->geometry());
    ct->show();
}
```

3.在关闭按键槽函数中写入以下代码实现关闭

```c++
void ctrl::xxxx_Solts()
{
    this->close();
}
```


# 正则表达式与验证器

通过使用验证器可以实现在输入框只输入特定的文本，不符合正则表达式的文本将不会显示。

## QT5版本

在Qt中，可以使用`QRegExp`类和`QValidator`类来判断`QLineEdit`中的值是否为数字。

方法一：使用QRegExp

`QRegExp`类提供了一种进行正则表达式匹配的方法，可以用于检测输入的内容是否符合指定的格式。我们可以使用`QRegExp`类实现输入验证，检测`QLineEdit`中的值是否为数字，例如：

```c++
QRegExp regExp("[0-9]*"); // 创建正则表达式，表示只接受数字
QRegExpValidator *validator = new QRegExpValidator(regExp, this); // 创建验证器
lineEdit->setValidator(validator); // 设置验证器
```

在这个例子中，我们创建了一个`QRegExp`对象，并指定其匹配规则为只接受数字。然后我们再创建了一个`QRegExpValidator`对象，并将其传递给`QLineEdit`对象的`setValidator()`函数中，这样可以保证`QLineEdit`中的值只能为数字。

方法二：使用QDoubleValidator

除了`QRegExpValidator`，Qt还提供了一种特定类型的验证器`QDoubleValidator`，可以用于检测输入的内容是否为浮点数。我们可以使用`QDoubleValidator`实现输入验证，检测`QLineEdit`中的值是否为数字，例如：

```c++
QDoubleValidator *validator = new QDoubleValidator(this); // 创建验证器
lineEdit->setValidator(validator); // 设置验证器
```

在这个例子中，我们创建了一个`QDoubleValidator`对象，并传递给`QLineEdit`对象的`setValidator()`函数中，这样可以保证`QLineEdit`中的值只能为浮点数。

无论是方法一还是方法二，都可以实现`QLineEdit`中的值是否为数字的输入验证功能。根据你的实际需求和场景特点，选择适合的方法即可。

## QT6版本

在Qt6中，`QRegExp`和`QRegExpValidator`已经被标记为已弃用，并且不再建议使用它们进行输入验证。

在Qt6中，可以使用`QRegularExpression`和`QRegularExpressionValidator`进行正则表达式匹配和输入验证。以下是使用`QRegularExpression`和`QRegularExpressionValidator`检测`QLineEdit`中的值是否为数字的示例代码：

```c++
QRegularExpression regExp("[0-9]*"); // 创建正则表达式，表示只接受数字
QRegularExpressionValidator *validator = new QRegularExpressionValidator(regExp, this); // 创建验证器
lineEdit->setValidator(validator); // 设置验证器
```

在使用`QRegularExpressionValidator`时，它会自动将`QLineEdit`的前景色设置为红色，并弹出一个小型的文本提示，以指示输入的值不符合规则。

由于Qt6与Qt5在某些细节方面略有不同，因此建议在编写新应用程序时首选Qt6的API，以便确保最大程度的兼容性和可靠性。

## 正则表达式

比如匹配小数，可以使用 `[0-9]*(\\.[0-9]*)?`

- `[0-9]*` 表示匹配 0 到多个数字，其中方括号 `[]` 中的字符为可以匹配的字符集合，`0-9` 表示匹配 0 到 9 的任意数字，`*` 表示匹配 0 到多个该字符集合中的字符。
- `(\\.[0-9]*)?` 表示一个可选的小数部分，其中的第一个括号 `()` 表示这是一个组，并且 `\\.` 表示匹配小数点，因为小数点是一个特殊的字符，需要用反斜线转义来表示；`[0-9]*` 同上，表示匹配 0 到多个数字，`?` 表示匹配 0 到 1 个该字符。
- 整个正则表达式最外层的方括号 `[]` 表示匹配一个字符集合，其中的 `*` 表示匹配 0 到多个字符。因此整个正则表达式表示匹配 0 到多个数字，加上一个可选的小数部分。

综合来说，这个正则表达式可以匹配如下字符串形式：

- `123`：只包含整数部分；
- `0`：只包含整数部分；
- `123.456`：包含小数部分和整数部分；
- `123.`：包含整数部分，但是小数点后面为空；
- `0.123`：小数部分前面为 0。

需要注意的是，这个正则表达式并不是万能的，它只能匹配一般的数字形式。如果你有特殊的匹配需求，可能需要根据实际情况编写更加复杂的正则表达式。

***
# 串口相关组件

## 添加串口相关头文件

在QT中使用串口组件，首先需要在项目.pro文件中添加一句`QT += serialport`，保存后会将相关文件添加进入项目中，同时在.h文件中引入下面几个头文件

分别是 

**#include <QSerialPort>** 

**#include <QSerialPortInfo>**

QSerialPort是关于串口本身设置相关的头文件，例如修改停止位，数据位，校验位之类的

QSerialPortInfo是用于收集串口相关数据所使用，例如获取当前可用串口号

## 获取可用串口号

```c++
/*读取可使用的串口号*/
QStringList serialNamePort;
serialPort = new QSerialPort(this);
foreach (const QSerialPortInfo &info, QSerialPortInfo::availablePorts()) { //将可用的端口放入info中
    serialNamePort << info.portName();      //将端口名传给QStringList
}
ui->UartCB->addItems(serialNamePort);       //输出到串口号CB上
```

上述程序就是将可用串口号存入serialNamePort中，再将内容添加到对应的ComboBox中，该段程序可以放在窗口的构造函数中

## 串口开启配置

```c++

void MainWindow::on_OpenPB_clicked()
{
    //如果串口已经打开，先清除并关闭
    if(serialPort->isOpen())
    {
        serialPort->clear();
        serialPort->close();
    }
    QSerialPort::BaudRate baudRate = QSerialPort::Baud115200;
    QSerialPort::DataBits dataBits = QSerialPort::Data8;
    QSerialPort::StopBits stopBits = QSerialPort::OneStop;
    QSerialPort::Parity checkBits  = QSerialPort::NoParity;

    //获取波特率控件上面的数据
    if(ui->BaudCB->currentText() == "9600") baudRate = QSerialPort::Baud9600;
    else if (ui->BaudCB->currentText() == "115200") baudRate = QSerialPort::Baud115200;

    //获取数据位控件上面的数据
    if(ui->DataCB->currentText() == "5") dataBits = QSerialPort::Data5;
    else if(ui->DataCB->currentText() == "6") dataBits = QSerialPort::Data6;
    else if(ui->DataCB->currentText() == "7") dataBits = QSerialPort::Data7;
    else if(ui->DataCB->currentText() == "8") dataBits = QSerialPort::Data8;

    //获取停止位控件上面的数据
    if(ui->StopCB->currentText() == "1") stopBits = QSerialPort::OneStop;
    else if(ui->StopCB->currentText() == "1.5") stopBits = QSerialPort::OneAndHalfStop;
    else if(ui->StopCB->currentText() == "2") stopBits = QSerialPort::TwoStop;

    //获取校验位控件上面的数据
    if(ui->checkCB->currentText() == "None") checkBits = QSerialPort::NoParity;
    else if(ui->checkCB->currentText() == "Odd") checkBits = QSerialPort::OddParity;
    else if(ui->checkCB->currentText() == "Even") checkBits = QSerialPort::EvenParity;

    //为串口赋值
    serialPort->setPortName(ui->UartCB->currentText());
    serialPort->setBaudRate(baudRate);
    serialPort->setDataBits(dataBits);
    serialPort->setStopBits(stopBits);
    serialPort->setParity(checkBits);
    if(serialPort->open(QIODevice::ReadWrite)== true)
    {
        QMessageBox::information(this,"提示",ui->UartCB->currentText()+"打开成功");
    }
    else
    {
        QMessageBox::critical(this,"提示",ui->UartCB->currentText()+"打开失败");
    }
}
```

开启按键的槽函数，有相关设置的配置，可以参考修改

## 串口接收槽函数

当串口接收到数据后，我们需要继续解析数据并做出相应操作，需要写一个槽函数。

==注意：在窗口的构造函数或者开启槽函数中需要使用关联函数==，例如`connect(serialPort,SIGNAL(readyRead()),this,SLOT(serialPortReadReady_Slot()));`

readyRead( )函数是串口组件的信号，当可以读取时，就发射这个信号，槽函数我这边叫做serialPortReadReady_Slot( )，下面是模板，可以自行添加功能

```c++
void MainWindow::serialPortReadReady_Slot()
{
    QString buf;
    buf = QString(serialPort->readLine());
    ui->ReceivePTE->ensureCursorVisible(); //通过滚动文本编辑确保光标可见,始终显示最新一行
    ui->ReceivePTE->insertPlainText(buf);
    
    //这里可以写一些数据简析，数据库也行，要注意处理速度
}
```



***
# SQLite数据库

## 添加SQLite相关头文件

在QT中使用串口组件，首先需要在项目.pro文件中添加一句`QT += sql`，保存后会将相关文件添加进入项目中，同时在.h文件中引入下面几个头文件

分别是

**#include <QSqlDatabase>** 通过这个类添加 / 删除 / 复制 / 关闭数据库实例
**#include <QSqlQuery>**数据库查询类
**#include <QSqlQueryModel>**执行 SQL 语句和遍历结果集的高级接口。可以用来为视图类 (如 QTableView) 提供数据。
**#include <QSqlError>**数据操作失败可以通过这个类获取相关的错误信息。

## 在QT中使用Sqlite数据库

- QSqlDatabase建立Qt应用程序和数据库的连接

```c++
//添加数据库驱动
db = QSqlDatabase::addDatabase("QSQLITE");
//设置数据库名字
db.setDatabaseName("menu.db");
//打开数据库
db.open();

```

<font color=red>Tips:</font>在使用Qt数据库模块需要在工程文件中添加"QT+=sql"

- QSqlQuery执行数据库操作的SQL语句

```c++
    QSqlQuery query;
    query.exec("SELECT\DELETE\INSERT\UPDATE等SQL语句");
```

- QSqlQueryModel获取结果集

```c++
QString str = QString("SELECT * FROM 表名");
QSqlQuery *model = new QSqlQueryModel;
model->setQuery(str);//执行查询操作，并将结果集保存到model对象中
ui->menuTableView->setModel(model);//显示查询结果

```
## SQLite创建表

```sqlite
CREATE TABLE database_name.table_name(
   column1 datatype  PRIMARY KEY(one or more columns),
   column2 datatype,
   column3 datatype,
   .....
   columnN datatype,
);
```

实例：

```sqlite
sqlite> CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);
```

这里涉及到了数据类型的问题，下表为SQLite所有的数据类型。

| **存储类** | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| NULL       | 值是一个 NULL 值。                                           |
| INTEGER    | 值是一个带符号的整数，根据值的大小存储在 1、2、3、4、6 或 8 字节中。 |
| REAL       | 值是一个浮点值，存储为 8 字节的 IEEE 浮点数字。              |
| TEXT       | 值是一个文本字符串，使用数据库编码（UTF-8、UTF-16BE 或 UTF-16LE）存储。 |
| BLOB       | 值是一个 blob 数据，完全根据它的输入存储。                   |

## SQLite常用语句

### Insert 语句

```sqlite
INSERT INTO TABLE_NAME [(column1, column2, column3,...columnN)]  
VALUES (value1, value2, value3,...valueN);
```
<font color=red>Tips:</font>当输入值和列表顺序一致时，可以不输入[ ]中的内容

### Select 语句

```sqlite
SELECT column1, column2, columnN FROM table_name;
```

如果想获取所有可用的字段

```sqlite
SELECT * FROM table_name;
```

### Update 语句

```sqlite
UPDATE table_name
SET column1 = value1, column2 = value2...., columnN = valueN
WHERE [condition]; //可以使用 AND 或 OR 运算符来结合 N 个数量的条件。
```

### Delete 语句

```sqlite
DELETE FROM table_name
WHERE [condition]; //可以使用 AND 或 OR 运算符来结合 N 个数量的条件。
```

### Like语句

SQLite 的 **LIKE** 运算符是用来匹配通配符指定模式的文本值。如果搜索表达式与模式表达式匹配，LIKE 运算符将返回真（true），也就是 1。这里有两个通配符与 LIKE 运算符一起使用：

- 百分号 （%）
- 下划线 （_）

```sqlite
SELECT column_list 
FROM table_name
WHERE column LIKE 'XXXX%'  --百分号（%）代表零个、一个或多个数字或字符。下划线（_）代表一个单一的数字或字符。这些符号可以被组合使用。
```

## Table View控件的使用

Table View控件一般用来显示SQLite数据库的数据，QTableView与QSqlTableModel绑定使用

为了显示更加美观，有以下几个函数可以实现相应的美化（其中JY901TB为Table View名字）

```c++
ui->JY901TB->verticalHeader()->setHidden(true);//把QTableView中第一列的默认数字列去掉
ui->JY901TB->horizontalHeader()->setHidden(true);//把QTableView中第一行表头去掉

//设置列宽，第一个参数为列号，第二个参数为宽度，可以自己试验
ui->JY901TB->setColumnWidth(0,50);
ui->JY901TB->setColumnWidth(1,75);
ui->JY901TB->setColumnWidth(2,100);
ui->JY901TB->setColumnWidth(3,100);

ui->JY901TB->resizeColumnsToContents();//将列宽自适应数据长度
ui->JY901TB->resizeRowsToContents();//将行宽自适应数据长度

ui->JY901TB->setAlternatingRowColors(true);//QTableView隔行换色
```

## Table Widget控件的使用

Table Widget是我在Crucis上位机多线程遇到困难时选择的第二方法，舍弃SQLite，转而专注于实时显示。Table Widget不能使用model进行统一读取。

实测在Crucis中，Table Widget并不是一个很好的选择，需要嵌套判断。

> [Qt QTableWidget及基本操作（详解版） (biancheng.net)](http://c.biancheng.net/view/1863.html)
>
> [tableWidget用法_成魔的羔羊的博客-CSDN博客](https://blog.csdn.net/qq_35040828/article/details/70208240)

## Table View与Table Widget的差异

|         区别点          |            QTableView            |                         QTableWidget                         |
| :---------------------: | :------------------------------: | :----------------------------------------------------------: |
|        继承关系         |                                  |                 QTableWidget继承自QTableView                 |
|  使用数据模型setModel   |   可以使用setModel设置数据模型   |        setModel是私有函数，不能使用该函数设置数据模型        |
| 显示复选框setCheckState |        没有函数实现复选框        | QTableWidgetItem类中的setCheckState(Qt::Checked);可以设置复选框 |
|  与QSqlTableModel绑定   | QTableView能与QSqlTableModel绑定 |             QtableWidget不能与QSqlTableModel绑定             |



***

# 多线程编程

在开发STM32的时候已经见识过多线程对于整个系统灵敏度的改变，在处理复杂任务时会比裸机快很多。那么在开发应用程序时，如果应用程序需要处理大量复杂的数据，仅仅使用主线程是远远不够的，会造成整个程序的卡顿甚至是未响应。例如大量数据的计算或者读写数据库的任务可以交给其他线程完成，最后再由主线程显示结果。

在QT中就有多线程的API，当然以下需要注意：

- 默认的线程在Qt中称之为窗口线程，也叫**主线程**，负责窗口事件处理或者窗口控件数据的更新
- 子线程负责后台的业务逻辑处理，子线程中不能对窗口对象做任何操作，这些事情需要交给窗口线程处理
- 主线程和子线程之间如果要进行数据的传递，需要使用Qt中的信号槽机制

目前创建多线程有两种方式，一种是**继承QThread**，另一种是**继承QObject**。接下来详细说明这两种创建多线程的流程，还有两种方式的对比。最后还会介绍一种基于线程池处理多任务的方式。

## 常用函数

### 常用公用成员函数

```c++
// QThread 类常用 API
// 构造函数
QThread::QThread(QObject *parent = Q_NULLPTR);
// 判断线程中的任务是不是处理完毕了
bool QThread::isFinished() const;
// 判断子线程是不是在执行任务
bool QThread::isRunning() const;

// Qt中的线程可以设置优先级
// 得到当前线程的优先级
Priority QThread::priority() const;
void QThread::setPriority(Priority priority);
优先级:
    QThread::IdlePriority         --> 最低的优先级
    QThread::LowestPriority
    QThread::LowPriority
    QThread::NormalPriority
    QThread::HighPriority
    QThread::HighestPriority
    QThread::TimeCriticalPriority --> 最高的优先级
    QThread::InheritPriority      --> 子线程和其父线程的优先级相同, 默认是这个
// 退出线程, 停止底层的事件循环
// 退出线程的工作函数
void QThread::exit(int returnCode = 0);
// 调用线程退出函数之后, 线程不会马上退出因为当前任务有可能还没有完成, 调回用这个函数是
// 等待任务完成, 然后退出线程, 一般情况下会在 exit() 后边调用这个函数
bool QThread::wait(unsigned long time = ULONG_MAX);
```

### 信号槽

```c++
// 和调用 exit() 效果是一样的
// 代用这个函数之后, 再调用 wait() 函数
[slot] void QThread::quit();
// 启动子线程
[slot] void QThread::start(Priority priority = InheritPriority);
// 线程退出, 可能是会马上终止线程, 一般情况下不使用这个函数
[slot] void QThread::terminate();

// 线程中执行的任务完成了, 发出该信号
// 任务函数中的处理逻辑执行完毕了
[signal] void QThread::finished();
// 开始工作之前发出这个信号, 一般不使用
[signal] void QThread::started();
```

### 静态函数

```c++
// 返回一个指向管理当前执行线程的QThread的指针
[static] QThread *QThread::currentThread();
// 返回可以在系统上运行的理想线程数 == 和当前电脑的 CPU 核心数相同
[static] int QThread::idealThreadCount();
// 线程休眠函数
[static] void QThread::msleep(unsigned long msecs);	// 单位: 毫秒
[static] void QThread::sleep(unsigned long secs);	// 单位: 秒
[static] void QThread::usleep(unsigned long usecs);	// 单位: 微秒

```

## 继承QThread创建线程

1. 需要创建一个线程类的子类，让其继承QThread

    ```c++
    class MyThread:public QThread
    {
        ......
    }
    ```

2. 重写父类的虚函数run()，在该函数内部写子线程需要完成的任务

    ```c++
    class MyThread:public QThread
    {
        ......
     protected:
        void run()
        {
            ........
        }
    }
    ```

3. 在主线程中创建子线程对象，new一个对象即可

    ```c++
    MyThread * subThread = new MyThread;
    ```

4. 启动子线程，调用start()方法

    ```c++
    subThread->start();
    ```

    <font color=red>Tips:</font>

    - run()函数是一个受保护的成员函数，不能够在类的外部调用。通过当前线程对象调用槽函数 start() 启动子线程，当子线程被启动，这个 run() 函数也就在线程内部被调用了。
    - 在 Qt 中在子线程中不要操作程序中的窗口类型对象，不允许，如果操作了程序就挂了
    - 只有主线程才能操作程序中的窗口对象，默认的线程就是主线程，自己创建的就是子线程

>案例八使用了该方法，具体代码在相关案例中（案例八九十要求相同）
## 继承QObject创建线程

1. 创建一个新的类，让这个类从 QObject 派生

    ```c++
    class MyWork:public QObject
    {
        .......
    }
    ```

2. 在这个类中添加一个公共的成员函数，函数体就是我们要子线程中执行的业务逻辑

    ```c++
    class MyWork:public QObject
    {
    public:
        .......
        // 函数名自己指定, 叫什么都可以, 参数可以根据实际需求添加
        void working();
    }
    ```

3. 在主线程中创建一个 QThread 对象，这就是子线程的对象

    `QThread* sub = new QThread;`

4. 在主线程中创建工作的类对象（千万不要指定给创建的对象指定父对象）

    ```c++
    MyWork* work = new MyWork(this);    // error
    MyWork* work = new MyWork;          // ok
    ```

5. 将 MyWork 对象移动到创建的子线程对象中，需要调用 QObject 类提供的 moveToThread() 方法

    ```c++
    // void QObject::moveToThread(QThread *targetThread);
    // 如果给work指定了父对象, 这个函数调用就失败了
    // 提示： QObject::moveToThread: Cannot move objects with a parent
    work->moveToThread(sub);	// 移动到子线程中工作
    ```

6. 启动子线程，调用 start(), 这时候线程启动了，但是移动到线程中的对象并没有工作

7. 调用 MyWork 类对象的工作函数，让这个函数开始执行，这时候是在移动到的那个子线程中运行的

    >案例九使用了该方法，具体代码在相关案例中（案例八九十要求相同）

## 线程资源释放

如果不使用线程池处理多任务，需要在最后对线程资源进行释放。举例如下：

```c++
connect(this,&MainWindow::destroyed,this,[=]()
{
    t1->quit();
    t1->wait();
    t1->deleteLater(); //或者delete t1;
});
```

## 两种多线程方式对比

两种多线程方式都可以完成任务，但是有区别。QT官方更加推荐继承QObject的方法，因为更加灵活。比如一个线程想要进行多个任务操作时，如果使用**继承QThread**的方法，需要重写run函数，写入所有任务，并且在后续更改中也十分麻烦。而采用**继承QObject**的方法可以自定好几个任务对象，自由将不同任务放入线程对象中，后续维护程序更加容易。

## 基于线程池处理多任务

线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。线程池线程都是后台线程。每个线程都使用默认的堆栈大小，以默认的优先级运行，并处于多线程单元中。如果某个线程在托管代码中空闲（如正在等待某个事件）, 则线程池将插入另一个辅助线程来使所有处理器保持繁忙。如果所有线程池线程都始终保持繁忙，但队列中包含挂起的工作，则线程池将在一段时间后创建另一个辅助线程但线程的数目永远不会超过最大值。超过最大值的线程可以排队，但他们要等到其他线程完成后才启动。

### QRunnable

在 Qt 中使用线程池需要先创建任务，添加到线程池中的每一个任务都需要是一个 QRunnable 类型，因此在程序中需要创建子类继承 QRunnable 这个类，然后重写 run() 方法，在这个函数中编写要在线程池中执行的任务，并将这个子类对象传递给线程池，这样任务就可以被线程池中的某个工作的线程处理掉了。

```c++
// 在子类中必须要重写的函数, 里边是任务的处理流程
[pure virtual] void QRunnable::run();

// 参数设置为 true: 这个任务对象在线程池中的线程中处理完毕, 这个任务对象就会自动销毁
// 参数设置为 false: 这个任务对象在线程池中的线程中处理完毕, 对象需要程序猿手动销毁
void QRunnable::setAutoDelete(bool autoDelete);
// 获取当前任务对象的析构方式,返回true->自动析构, 返回false->手动析构
bool QRunnable::autoDelete() const;
```

使用QRunnable实例

```c++
class MyWork : public QObject, public QRunnable
{
    Q_OBJECT
public:
    explicit MyWork(QObject *parent = nullptr)
    {
        // 任务执行完毕,该对象自动销毁
        setAutoDelete(true);
    }
    ~MyWork();

    void run() override{}
}
```

<font color=red>Tips:</font>如果不使用QT中的信号槽机制，可以不继承QOject

### QThreadPool

Qt 中的 QThreadPool 类管理了一组 QThreads, 里边还维护了一个任务队列。QThreadPool 管理和回收各个 QThread 对象，以帮助减少使用线程的程序中的线程创建成本。每个Qt应用程序都有一个全局 QThreadPool 对象，可以通过调用 globalInstance() 来访问它。也可以单独创建一个 QThreadPool 对象使用。

```c++
// 获取和设置线程中的最大线程个数
int maxThreadCount() const;
void setMaxThreadCount(int maxThreadCount);

// 给线程池添加任务, 任务是一个 QRunnable 类型的对象
// 如果线程池中没有空闲的线程了, 任务会放到任务队列中, 等待线程处理
void QThreadPool::start(QRunnable * runnable, int priority = 0);
// 如果线程池中没有空闲的线程了, 直接返回值, 任务添加失败, 任务不会添加到任务队列中
bool QThreadPool::tryStart(QRunnable * runnable);

// 线程池中被激活的线程的个数(正在工作的线程个数)
int QThreadPool::activeThreadCount() const;

// 尝试性的将某一个任务从线程池的任务队列中删除, 如果任务已经开始执行就无法删除了
bool QThreadPool::tryTake(QRunnable *runnable);
// 将线程池中的任务队列里边没有开始处理的所有任务删除, 如果已经开始处理了就无法通过该函数删除了
void QThreadPool::clear();

// 在每个Qt应用程序中都有一个全局的线程池对象, 通过这个函数直接访问这个对象
static QThreadPool * QThreadPool::globalInstance();
```

使用QThreadPool实例

```c++
MyWork::MyWork() : QRunnable()
{
    // 任务执行完毕,该对象自动销毁
    setAutoDelete(true);
}
void MyWork::run()
{
    // 业务处理代码
    ......
}
```

最后在mainwindow中的使用实例

```c++
MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    // 线程池初始化，设置最大线程池数
    QThreadPool::globalInstance()->setMaxThreadCount(4);
    // 添加任务
    MyWork* task = new MyWork;
    QThreadPool::globalInstance()->start(task);    
}
```

> 案例十使用了该方法，具体代码在相关案例中（案例八九十要求相同）

## 串口多线程

使用QThread方法

```c++
//主UI内
		//启动数据接收线程
        qDebug() << "外部serial" << serial;
        SRDThread = new SerialReadData(serial);
        SRDThread->start();
        connect(serial,&QSerialPort::readyRead,SRDThread,[=](){
            SRDThread->m_startRequested = true;
        });

//线程类中
SerialReadData::SerialReadData(QSerialPort *serial,QObject *parent)
    : QThread{parent}
{
    Q_UNUSED(parent);
    qDebug() << "构造serial" <<serial;
    m_pserial = serial;
    qDebug() << "串口复制" << m_pserial;
}

void SerialReadData::run()
{
    while(1)
    {
        if(m_startRequested)
        {
            serialBuf = QString(m_pserial->readLine());
            if(!serialBuf.isEmpty())
            {
                qDebug() << serialBuf;
                QStringList ProcessedData = serialBuf.split(u' ');
                qDebug() << ProcessedData;
                //发射信号给datadisplay窗口
                emit uiDataDisplay(serialBuf);
                emit dataSort(serialBuf);
            }
            m_startRequested = false;
        }
    }
    exec();
}
```

使用QObject方法

```c++
//主UI内
		//启动数据接收线程
        qDebug() << "外部serial" << serial;

        QThread *t1 = new QThread;
        SRDThread = new SerialReadData(serial);
        SRDThread->moveToThread(t1);
        t1->start();
        connect(serial,&QSerialPort::readyRead,SRDThread,&SerialReadData::working);

//线程类中
SerialReadData::SerialReadData(QSerialPort *serial,QObject *parent)
    : QObject{parent}
{
    Q_UNUSED(parent);
    qDebug() << "构造serial" <<serial;
    m_pserial = serial;
    qDebug() << "串口复制" << m_pserial;
}

void SerialReadData::working()
{
    serialBuf = QString(m_pserial->readLine());
    if(!serialBuf.isEmpty())
    {
        qDebug() << serialBuf;
        QStringList ProcessedData = serialBuf.split(u' ');
        qDebug() << ProcessedData;
        //发射信号给datadisplay窗口
        emit uiDataDisplay(serialBuf);
        emit dataSort(serialBuf);
    }
}
```



***

# 手柄控制

## QGamepad

QGamepad只能在QT5中使用，是QT内部的一个模块，可以非常方便的控制手柄。

优点：

1. 非常方便的使用，只需要在.pro文件中加上`QT += gamepad`就可以使用相关函数
2. 连接简单，一句函数就能成功

缺点：
1. 每个按键都需要单独设置一个槽函数，比较麻烦
2. 只能在QT5上使用，QT6需要使用下面的QJoysticks

```C++
MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    //连接设备
    QGamepad *m_gamepad = new QGamepad(0, this);
    qDebug() << m_gamepad->deviceId();

    // 连接手柄的axisLeftXChanged()和axisLeftYChanged()信号，每次信号触发时会输出手柄的X和Y轴模拟量
    QObject::connect(m_gamepad, &QGamepad::axisLeftXChanged, [](double value) {
        //qDebug() << "X:" << value;
        LeftX = value;
    });

    QObject::connect(m_gamepad, &QGamepad::axisLeftYChanged, [](double value) {
        //qDebug() << "Y:" << value;
        LeftY = value;
    });

    // 创建一个定时器，每500ms执行一次
    QTimer *timer = new QTimer(this);
    timer->setInterval(500);

    connect(timer, &QTimer::timeout,this,[](){
        if(LeftX != 0)
            qDebug() << "X:" << LeftX;
        if(LeftY != 0)
            qDebug() << "Y:" << LeftY;
    });

    // 启动定时器和事件循环
    timer->start();

//    connect(m_gamepad, &QGamepad::axisLeftXChanged, this, [](double value){
//        qDebug() << "Left X" << value;
//    });
//    connect(m_gamepad, &QGamepad::axisLeftYChanged, this, [](double value){
//        qDebug() << "Left Y" << value;
//    });
//    connect(m_gamepad, &QGamepad::axisRightXChanged, this, [](double value){
//        qDebug() << "Right X" << value;
//    });
//    connect(m_gamepad, &QGamepad::axisRightYChanged, this, [](double value){
//        qDebug() << "Right Y" << value;
//    });
    connect(m_gamepad, &QGamepad::buttonAChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_A->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_A->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button A" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonBChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_B->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_B->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button B" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonXChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_X->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_X->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button X" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonYChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_Y->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_Y->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button Y" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonL1Changed, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_L1->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_L1->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button L1" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonR1Changed, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_R1->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_R1->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button R1" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonL2Changed, this, [](double value){
        qDebug() << "Button L2: " << value;
    });
    connect(m_gamepad, &QGamepad::buttonR2Changed, this, [](double value){
        qDebug() << "Button R2: " << value;
    });
    connect(m_gamepad, &QGamepad::buttonSelectChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_SELECT->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_SELECT->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button Select" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonStartChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_START->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_START->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "Button Start" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonGuideChanged, this, [](bool pressed){
        qDebug() << "Button Guide" << pressed;
    });

    connect(m_gamepad, &QGamepad::buttonUpChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_UP->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_UP->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "buttonUp" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonDownChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_DOWN->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_DOWN->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "buttonDown" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonLeftChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_LEFT->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_LEFT->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "buttonLeft" << pressed;
    });
    connect(m_gamepad, &QGamepad::buttonRightChanged, this, [this](bool pressed){
        switch (pressed) {
            case true:
                this->ui->BT_RIGHT->setStyleSheet(BTN_color);
                //qDebug() << "Button Atrue:" ;
                break;
            case false:
                this->ui->BT_RIGHT->setStyleSheet(BTN_color2);
                //qDebug() << "Button Afalse:" ;
            break;
        }
        //qDebug() << "buttonRight" << pressed;
    });
}
```

## QJoysticks

QJoysticks是一个第三方手柄控制库，源码位于[alex-spataru/QJoysticks: Joystick input library for Qt (github.com)](https://github.com/alex-spataru/QJoysticks)

使用步骤：

1. 克隆该库，并且放入工程文件夹中
2. 在.pro文件中加入include(Qjoysticks/Qjoysticks.pri)
3. 在头文件中\#include "QJoysticks.h"
4. 在cpp文件中m_joystick = QJoysticks::getInstance();获取当前可用手柄对象

```c++
MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    m_joystick = QJoysticks::getInstance();

    connect(m_joystick, SIGNAL(axisChanged(int, int, qreal)), this, SLOT(joysitck_axis(int, int, qreal)));
    //connect(m_joystick, &QJoysticks::axisEvent, this, &MainWindow::event_axis);

    connect(m_joystick, SIGNAL(buttonChanged(int, int, bool)), this, SLOT(joysitck_button(int, int, bool)));
    //connect(m_joystick, &QJoysticks::buttonEvent, this, &MainWindow::event_button);
}

void MainWindow::scan_joysticks()
{
    m_joystick->setVirtualJoystickEnabled (true);    //启用虚拟手柄
    m_joystick->setVirtualJoystickRange (1);    //设置虚拟手柄摇杆范围[-1,1]
    QStringList js_names = m_joystick->deviceNames();    //添加手柄
    qDebug() << js_names;
}

void MainWindow::joysitck_axis(int js_index, int axis_index, qreal value)
{
    if (m_joystick->joystickExists(js_index)){
        //左右
        if (axis_index == 0){
            // axis range [-1, 1] -> range [0, 1000];
            qDebug() << "x" << value*500+500;
        }
        //上下
        if (axis_index == 1){
            // axis range [-1, 1] -> range [0, 1000];
            qDebug() << "y" << value*500+500;
        }

    }
}

void MainWindow::event_axis(const QJoystickAxisEvent &event)
{
    qDebug() << "axis" << event.axis << "value" << event.value;
}

void MainWindow::joysitck_button(int js_index, int button_index, bool pressed)
{
    if (m_joystick->joystickExists(js_index)){
        qDebug() << button_index << pressed;
    }
}

void MainWindow::event_button(const QJoystickButtonEvent &event)
{
    qDebug() << "button" <<event.button << "pressed" << event.pressed;
}
```



# 打包和部署

完成上位机的编写后需要打包给其他人使用，需要经过更改图标，打包封装的过程。

## 更改图标

1. 将工程切换到<font color = red>release模式</font>，编译。

2. 去网上下载.ico的图标，添加到工程目录下。

3. 更改.pro文件，添加代码RC_ICONS = 文件名。

4. 再次编译，在工程目录下的release文件夹中，exe文件的图标完成了更改。

如果碰到以下类似报错：==[Makefile.Release:75:release/CrucisIPC_resource_res.o] Error 1==

有两种情况：

1. 你的图片icon，并不是真的.ico文件 ，比如jpg文件或者png文件，你为了方便偷偷的把它的后缀名字改成了.ico文件，说，你是不是这样干的（别问我咋知道的╮(￣▽￣")╭）。附带转换网站   [在线转换图片格式-在线图片转换器- JinaConvert.com](https://jinaconvert.com/cn/index.php)

2. 路径问题，一般和.pro文件放在一起，当然也可以放在其他文件夹，但是要写路径。

> [:-1: error: [Makefile.Debug:72: debug/QtIcon_resource_res.o\] Error 1 原因与彻底解决方案_比卡丘不皮的博客-CSDN博客](https://blog.csdn.net/weixin_42126427/article/details/106686824)

## 打包封装

## 使用QT控制台

1. 使用QT的控制台，新建一个文件夹<font color=red>（不能含有中文）</font>，放置生成的包。

2. 将release中的exe文件拷贝到上述新建文件夹中。

3. 用QT控制台移动到文件夹中，指令为cd /d （路径）。

4. 使用指令windeployqt （exe文件名），封包完成。

    ![image-20221115134146071](QT开发.assets/image-20221115134146071.png)

    <center><p>cd的示例</p></center>

## 使用Windows Powershell

1. 使用mingw_64或者msvc2019_64切换到release模式进行编译

2. 新建一个文件夹<font color=red>（不能含有中文）</font>

3. 在文件夹中放入编译完成文件夹中的exe文件

4. （使用mingw_64编译有此步骤）将 `盘符:\QT\6.4.2\mingw_64\bin`中libgcc_s_seh-1.dll、libstdc++-6.dll、libwinpthread-1.dll三个动态库文件放入新建的文件夹中

5. 启动Windows Powershell ，使用cd命令切换到新建的文件夹的路径下

6. 执行 `盘符:\QT\6.4.2\mingw_64\bin\windeployqt.exe 文件名.exe` (使用mingw_64编译)

    或者`盘符:\QT\6.4.2\msvc2019_64\bin\windeployqt.exe 文件名.exe`(使用msvc2019_64编译)

目前测出QT6.4.3存在打包问题，使用6.4.2就可以正确打包

***

# 相关案例

## 案例一	滑块与选值框

创建应用程序，包含滑块（QSlider）和选值框（QSpinBox），通过信号和槽关联，保持同步运行

```c++
int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    Widget w;

    //创建水平滑块
    QSlider slider(Qt::Horizontal,&w);
    slider.move(20,100);

    //创建选值框
    QSpinBox spinbox(&w);
    spinbox.move(220,100);
    
    //滑块改变，选值框改变
    QObject::connect(&slider,SIGNAL(valueChanged(int)),&spinbox,SLOT(setValue(int)));
    //选值框改变，滑块改变
    QObject::connect(&spinbox,SIGNAL(valueChanged(int)),&slider,SLOT(setValue(int)));

    w.show();
    return a.exec();
}
```

## 案例二	计算器

- 输入两个数字，按“=”按钮显示计算结果。
- 只要两个数使合法数字时，“=”按钮才被激活，否则禁用
- 显示结果控件只可查看不可修改，但支持复制到剪切板
- 所以子窗口大小和位置随主窗口的缩放自动调整至最佳

```c++
/*calculator.h*/
#ifndef __CALCULATOR_H
#define __CALCULATOR_H

#include <QWidget>
#include <QLabel>
#include <QPushButton>
#include <QLineEdit>
#include <QHBoxLayout>      //水平布局器
#include <QDoubleValidator> //验证器

QT_BEGIN_NAMESPACE
namespace Ui { class Calculator; }
QT_END_NAMESPACE

class Calculator : public QWidget
{
    Q_OBJECT

public:
    Calculator(QWidget *parent = nullptr);
    ~Calculator();

public slots:
    //使能按键
    void enableButton(void);
    //计算结果和显示的槽函数
    void calcClinked(void);
private:
    Ui::Calculator *ui;
    /*三个line edit*/
    QLineEdit* m_editX; //左操作数
    QLineEdit* m_editY; //右操作数
    QLineEdit* m_editZ; //结果

    QLabel* m_label;    //"+"
    QPushButton* m_button;   //"="
};
#endif // CALCULATOR_H

```

```c++
/*calculator.cpp*/
#include "calculator.h"
#include "ui_calculator.h"

Calculator::Calculator(QWidget *parent)
    : QWidget(parent)
    , ui(new Ui::Calculator)
{
    ui->setupUi(this);

    //界面初始化
    setWindowTitle("计算器");
    //左操作数创建
    m_editX = new QLineEdit(this);
    //文本右对齐
    m_editX->setAlignment(Qt::AlignRight);
    //设置数字验证器，只能输入数字形式
    m_editX->setValidator(new QDoubleValidator(this));

    //右操作数
    m_editY = new QLineEdit(this);
    m_editY->setAlignment(Qt::AlignRight);
    m_editY->setValidator(new QDoubleValidator(this));

    //显示结果
    m_editZ = new QLineEdit(this);
    m_editZ->setAlignment(Qt::AlignRight);
    //只读属性
    m_editZ->setReadOnly(true);

    //"+"
    m_label = new QLabel("+",this);
    //“=”
    m_button = new QPushButton("=",this);
    m_button->setEnabled(false);//设置禁用
    //创建布局器
    QHBoxLayout* layout = new QHBoxLayout(this);
    //按水平方向加入布局器
    layout->addWidget(m_editX);
    layout->addWidget(m_label);
    layout->addWidget(m_editY);
    layout->addWidget(m_button);
    layout->addWidget(m_editZ);
    //设置布局器
    setLayout(layout);

    //信号与槽的链接
    connect(m_editX,SIGNAL(textChanged(QString)),this,SLOT(enableButton(void)));
    connect(m_editY,SIGNAL(textChanged(QString)),this,SLOT(enableButton(void)));

    connect(m_button,SIGNAL(clicked(bool)),this,SLOT(calcClinked()));

}

Calculator::~Calculator()
{
    delete ui;
}

void Calculator::enableButton(void)
{
    bool bxok,byok;
    //判断是否为数字，如果是数字将bool赋值true
    m_editX->text().toDouble(&bxok);
    m_editY->text().toDouble(&byok);

    m_button->setEnabled(bxok && byok);
}

void Calculator::calcClinked(void)
{
    //计算值
    double res = m_editX->text().toDouble() + m_editY->text().toDouble();
    //转换成字符串
    QString str = QString::number(res);
    //显示
    m_editZ->setText(str);
}

```

## 案例三	获取系统时间

- 使用QT创建获取时间窗口

- 点击按钮，通过槽函数获取时间并且显示在标签中

```c++
/*timewidget.h*/
#ifndef TIMEWIDGET_H
#define TIMEWIDGET_H

#include <QWidget>
#include <QLabel>
#include <QPushButton>
#include <QVBoxLayout>  //垂直布局器
#include <QTime>
#include <QDebug>   //打印调试
#include <QFont>
QT_BEGIN_NAMESPACE
namespace Ui { class TimeWidget; }
QT_END_NAMESPACE

class TimeWidget : public QWidget
{
    Q_OBJECT

public:
    TimeWidget(QWidget *parent = nullptr);
    ~TimeWidget();

public slots:
    void getTime(void); //获取时间槽函数

private:
    Ui::TimeWidget *ui;

    QLabel* m_label;        //显示时间label
    QPushButton* m_button;  //获取时间button
};
#endif // TIMEWIDGET_H
```

```c++
/*timewidget.cpp*/
#include "timewidget.h"
#include "ui_timewidget.h"

TimeWidget::TimeWidget(QWidget *parent)
    : QWidget(parent)
    , ui(new Ui::TimeWidget)
{
    ui->setupUi(this);

    //初始化界面

    //初始化label
    m_label = new QLabel(this);
    //设置label边框效果:凹陷面板
    m_label->setFrameStyle(QFrame::Panel|QFrame::Sunken);
    //设置label文本的对齐方式：水平|垂直居中
    m_label->setAlignment(Qt::AlignHCenter|Qt::AlignVCenter);
    //设置label文本字体大小
    QFont font;
    font.setPointSize(20);
    m_label->setFont(font);

    //初始化button
    m_button = new QPushButton("获取当前时间",this);
    m_button->setFont(font);

    //创建垂直布局器
    QVBoxLayout* layout = new QVBoxLayout(this);
    layout->addWidget(m_label);
    layout->addWidget(m_button);
    setLayout(layout);

    //信号与槽函数链接
    connect(m_button,SIGNAL(clicked(void)),this,SLOT(getTime(void)));
}

TimeWidget::~TimeWidget()
{
    delete ui;
}

void TimeWidget::getTime()
{
    qDebug("getTime");

    //获取系统时间
    QTime time = QTime::currentTime();
    //将时间对象转换为字符串
    QString str = time.toString("hh:mm:ss");
    //放入label中
    m_label->setText(str);
}
```

## 案例四 登录对话框

- 点击“OK”按钮判断输入用户名和密码是否正确，如果错误弹出消息提示栏，可以重新登陆
- 点击“Cancel”按钮，则退出登录

```c++
/* loginwiget.h */
#ifndef LOGINWIDGET_H
#define LOGINWIDGET_H

#include <QWidget>
#include <QMessageBox>
#include <QDebug>

QT_BEGIN_NAMESPACE
namespace Ui { class LoginWidget; }
QT_END_NAMESPACE

class LoginWidget : public QWidget
{
    Q_OBJECT

public:
    LoginWidget(QWidget *parent = nullptr);
    ~LoginWidget();
public slots:
    //处理ok按钮的槽函数
    void onAccepted(void);
    //处理cancel按钮的槽函数
    void onRejected(void);
private:
    Ui::LoginWidget *ui;
};
#endif // LOGINWIDGET_H

```

```c++
/* loginwidget.cpp */
#include "loginwidget.h"
#include "ui_loginwidget.h"


LoginWidget::LoginWidget(QWidget *parent)
    : QWidget(parent)
    , ui(new Ui::LoginWidget)
{
    //界面初始化
    ui->setupUi(this);
    //信号与槽函数的链接
    connect(ui->m_btnBox,SIGNAL(accepted(void)),this,SLOT(onAccepted(void)));
    connect(ui->m_btnBox,SIGNAL(rejected(void)),this,SLOT(onRejected(void)));
}

LoginWidget::~LoginWidget()
{
    delete ui;
}

void LoginWidget::onAccepted(void)
{
    //如果用户名等于bcl，密码为123 提示登录成功，否则提示失败
    if(ui->m_usernameLE->text() == "bcl" && ui->m_passwordLE->text() == "123")
    {
        qDebug("登陆成功！");
        close();
    }
    else
    {
        QMessageBox msgbox(QMessageBox::Critical,   //图标
                           "Error",     //标题
                           "用户名或者密码错误", //提示消息
                           QMessageBox::Ok,  //按钮
                           this);   //父窗口指针
        msgbox.exec();
    }
}

void LoginWidget::onRejected(void)
{
    QMessageBox msgbox(QMessageBox::Question,
                       "登陆",
                       "是否确定要取消登录？",
                       QMessageBox::Yes|QMessageBox::No,
                       this);
    if(msgbox.exec() == QMessageBox::Yes)
    {
        close();
    }
}
```

## 案例五 基于资源的图片浏览器

使用两个按键实现上一张和下一张切换

```c++
/* showimagewidget.h */
#ifndef SHOWIMAGEWIDGET_H
#define SHOWIMAGEWIDGET_H

#include <QWidget>
#include <QPainter>
#include <QImage>

QT_BEGIN_NAMESPACE
namespace Ui { class ShowImageWidget; }
QT_END_NAMESPACE

class ShowImageWidget : public QWidget
{
    Q_OBJECT

public:
    ShowImageWidget(QWidget *parent = nullptr);
    ~ShowImageWidget();

private slots:
    void on_m_btnPrev_clicked();

    void on_m_btnNext_clicked();

private:
    Ui::ShowImageWidget *ui;
    int m_index;//图片索引

    //绘图事件处理函数
    void paintEvent(QPaintEvent*);
};
#endif // SHOWIMAGEWIDGET_H

```

```c++
/* showimagewidget.cpp */
#include "showimagewidget.h"
#include "ui_showimagewidget.h"

ShowImageWidget::ShowImageWidget(QWidget *parent)
    : QWidget(parent)
    , ui(new Ui::ShowImageWidget)
{
    ui->setupUi(this);
    m_index = 0;
}

ShowImageWidget::~ShowImageWidget()
{
    delete ui;
}


void ShowImageWidget::on_m_btnPrev_clicked()
{
    if(--m_index < 0) m_index = 5;
    update();
}


void ShowImageWidget::on_m_btnNext_clicked()
{
    if(++m_index > 5) m_index = 0;
    update();
}

//绘图事件处理函数
void ShowImageWidget::paintEvent(QPaintEvent*)
{
    //创建画家对象
    QPainter painter(this);
    //获取绘图所在的矩形区域
    QRect rect = ui->frame->frameRect();
    //坐标值平移
    rect.translate(ui->frame->pos());
    //构建要绘制的图形对象
    QImage image(":/image/IMG_067"+QString::number(m_index)+".JPG");
    //使用painter将image画到rect中
    painter.drawImage(rect,image);
}

```

## 案例六 图片摇奖器

```c++
/* rafflewidget.h */
#ifndef RAFFLEWIDGET_H
#define RAFFLEWIDGET_H

#include <QWidget>
#include <QTimer>
#include <QPainter>
#include <QDir>
#include <QTime>
#include <QVector>
#include <QImage>
#include <QDebug>

QT_BEGIN_NAMESPACE
namespace Ui { class RaffleWidget; }
QT_END_NAMESPACE

class RaffleWidget : public QWidget
{
    Q_OBJECT

public:
    RaffleWidget(QWidget *parent = nullptr);
    ~RaffleWidget();

private slots:
    //按钮点击后的槽函数
    void on_pushButton_clicked();
    //定时器到时后执行的槽函数
    void onTimeOut(void);
private:
    Ui::RaffleWidget *ui;
    QVector<QImage> m_vecPhotos; //保存图片的容器
    int m_index;    //图片索引
    QTimer m_timer; //定时器对象
    bool isStarted; //标志位：true在摇奖；false停止摇奖
    //加载图片容器功能
    void loadPhotos(const QString& path);
    //定时器事件
    //void timerEvent(QTimerEvent*);
    //绘画事件
    void paintEvent(QPaintEvent*);
};
#endif // RAFFLEWIDGET_H

```

```c++
/* rafflewidget.cpp */
#include "rafflewidget.h"
#include "ui_rafflewidget.h"

RaffleWidget::RaffleWidget(QWidget *parent)
    : QWidget(parent)
    , ui(new Ui::RaffleWidget)
{
    ui->setupUi(this);
    m_index = 0;
    isStarted = false;

    //设计随机数种子
    srand(QTime::currentTime().msec());
    //加载所有的图片到容器中
    loadPhotos("./photos");
    qDebug() << "加载图片个数" << m_vecPhotos.size();
    connect(&m_timer,SIGNAL(timeout()),this,SLOT(onTimeOut()));
}

RaffleWidget::~RaffleWidget()
{
    delete ui;
}


void RaffleWidget::on_pushButton_clicked()
{
    if(isStarted == false)
    {
        isStarted = true;//摇奖开始
        m_timer.start(50);
        ui->pushButton->setText("停止");
    }
    else
    {
        isStarted = false;//摇奖结束
        m_timer.stop();
        ui->pushButton->setText("开始");
    }
}

//加载图片容器功能
void RaffleWidget::loadPhotos(const QString& path)
{
    QDir dir(path);
    //遍历当前目录的所有图片
    QStringList list1 = dir.entryList(QDir::Files);
    for (int i = 0; i < list1.size(); ++i) {
        QImage image(path+"/"+list1.at(i));
        m_vecPhotos << image;
    }
    //递归遍历子目录的图片
    QStringList list2 = dir.entryList(QDir::Dirs|QDir::NoDotAndDotDot);
    for (int i = 0; i < list2.size(); ++i) {
        loadPhotos(path+"/"+list2.at(i));
    }
}
//定时器事件
//void RaffleWidget::timerEvent(QTimerEvent*)
//{
//    m_index = rand() % m_vecPhotos.size();
//    update();
//}
//绘画事件
void RaffleWidget::paintEvent(QPaintEvent*)
{
    QPainter painter(this);
    QRect rect = ui->frame->frameRect();
    rect.translate(ui->frame->pos());
    painter.drawImage(rect,m_vecPhotos[m_index]);
}

void RaffleWidget::onTimeOut(void)
{
    m_index = rand() % m_vecPhotos.size();
    update();
}

```

## 案例七 学生成绩管理系统

```c++
/* studentdialog.h */
#ifndef STUDENTDIALOG_H
#define STUDENTDIALOG_H

#include <QDialog>
#include <QSqlDatabase>
#include <QSqlQuery>
#include <QSqlQueryModel>
#include <QSqlError>
#include <QDebug>
#include <QMessageBox>

QT_BEGIN_NAMESPACE
namespace Ui { class StudentDialog; }
QT_END_NAMESPACE

class StudentDialog : public QDialog
{
    Q_OBJECT

public:
    StudentDialog(QWidget *parent = nullptr);
    ~StudentDialog();
private:
    //创建数据库
    void createDB();
    //创建数据表
    void createTable();
    //查询
    void queryTable();
private slots:
    //插入按钮对应的槽函数
    void on_insertButton_clicked();
    //删除按钮对应的槽函数
    void on_deleteButton_clicked();
    //修改按钮对应的槽函数
    void on_editButton_clicked();
    //排序按钮对应的槽函数
    void on_sortButton_clicked();

private:
    Ui::StudentDialog *ui;
    QSqlDatabase db;//建立QT和数据库连接
    QSqlQueryModel model;//保存结果集
    bool DataExists = false;//存在数据判断标识
};
#endif // STUDENTDIALOG_H

```

```c++
/* studentdialog.cpp */
#include "studentdialog.h"
#include "ui_studentdialog.h"

StudentDialog::StudentDialog(QWidget *parent)
    : QDialog(parent)
    , ui(new Ui::StudentDialog)
{
    ui->setupUi(this);
    createDB();
    createTable();
    queryTable();
}

StudentDialog::~StudentDialog()
{
    delete ui;
}

//创建数据库
void StudentDialog::createDB()
{
    //添加数据库的驱动
    db = QSqlDatabase::addDatabase("QSQLITE");
    //设置数据库名字（文件名）
    db.setDatabaseName("student.db");
    //打开数据库
    if(db.open()==true)
    {
        qDebug() << "创建或者打开数据库成功！";
    }
    else
    {
        qDebug() << "创建或者打开数据库失败！";
    }

}
//创建数据表
void StudentDialog::createTable()
{
    QSqlQuery query;
    QString str = QString("CREATE TABLE student ("
                          "id INT PRIMARY KEY NOT NULL,"
                          "name TEXT NOT NULL,"
                          "score REAL NOT NULL)");
    if(query.exec(str) == false)
    {
        qDebug() << str;
    }
    else
    {
        qDebug() << "创建数据表成功！";
    }
}
//查询
void StudentDialog::queryTable()
{
    QString str = QString("SELECT * FROM student");
    model.setQuery(str);
    ui->tableView->setModel(&model);
}

//插入按钮对应的槽函数
void StudentDialog::on_insertButton_clicked()
{
	QSqlQuery query;
    //获取id name score
    int id = ui->IDEdit->text().toInt();
    if(id == 0) QMessageBox::critical(this,"Error","ID输入错误！");

    QString name = ui->nameEdit->text();
    if(name == "") QMessageBox::critical(this,"Error","姓名输入错误！");

    double score = ui->scoreEdit->text().toDouble();
    if(score<0 || score>100) QMessageBox::critical(this,"Error","成绩输入错误！");

    if(DataExists == true)  //存在数据执行删除语句
    {
        QString str = QString("DELETE FROM student WHERE id = %1").arg(id);
        query.exec(str);
        qDebug() << str;
        DataExists = false;
    }

    QString str = QString("INSERT INTO student VALUES(%1,'%2',%3)").arg(id).arg(name).arg(score);
    if(query.exec(str)==false)
    {
        qDebug() << str;
    }
    else
    {
        qDebug() << "插入数据成功！";
        DataExists = true;//表示已有数据
        queryTable();
    }

}

//删除按钮对应的槽函数
void StudentDialog::on_deleteButton_clicked()
{
    QSqlQuery query;
    //获取id
    int id = ui->IDEdit->text().toInt();

    QString str = QString("DELETE FROM student WHERE id = %1").arg(id);

    if(QMessageBox::question(this,"删除","确定要删除吗？",QMessageBox::Yes|QMessageBox::No) == QMessageBox::No)
    {
        return;
    }

    if(query.exec(str)==false)
    {
        qDebug() << str;
    }
    else
    {
        qDebug() << "删除数据成功！";
        queryTable();
    }
}

//删除按钮对应的槽函数
void StudentDialog::on_deleteButton_clicked()
{
    QSqlQuery query;
    //获取id
    int id = ui->IDEdit->text().toInt();

    QString str = QString("DELETE FROM student WHERE id = %1").arg(id);

    if(QMessageBox::question(this,"删除","确定要删除吗？",QMessageBox::Yes|QMessageBox::No) == QMessageBox::No)
    {
        return;
    }

    if(query.exec(str)==false)
    {
        qDebug() << str;
    }
    else
    {
        qDebug() << "删除数据成功！";
        queryTable();
    }
}

//修改按钮对应的槽函数
void StudentDialog::on_editButton_clicked()
{
    QSqlQuery query;

    //获取id score
    int id = ui->IDEdit->text().toInt();
    double score = ui->scoreEdit->text().toDouble();

    QString str = QString("UPDATE student SET score=%1 WHERE id =%2").arg(score).arg(id);
    if(query.exec(str)==false)
    {
        qDebug() << str;
    }
    else
    {
        qDebug() << "修改数据成功！";
        queryTable();
    }
}

//排序按钮对应的槽函数
void StudentDialog::on_sortButton_clicked()
{
    //获取排序列名
    QString value = ui->valueComboBox->currentText();
    //排序排序方式
    QString condition;
    if(ui->condComboBox->currentIndex()==0)
    {
        condition = "ASC";//升序
    }
    else
    {
        condition = "DESC";//降序
    }
    QString str = QString("SELECT * FROM student ORDER BY %1 %2").arg(value).arg(condition);
    //查询与显示
    model.setQuery(str);
    ui->tableView->setModel(&model);
}


```

## 案例八  QThread多线程

- 点击开始按钮，在线程A中生成100000个随机数
- 在线程B中对100000个数进行冒泡排序
- 在线程C中对100000个数进行快速排序
- 最后在主线程中显示

```c++
/*仅展示重要部分*/
//bubblesort.cpp
#include "bubblesort.h"

BubbleSort::BubbleSort(QObject *parent)
    : QThread{parent}
{

}

void BubbleSort::recvArray(QVector<int> list)
{
    m_list = list;
}

void BubbleSort::run()
{
    qDebug()<<"开始排序";
    time.start();
    bubblesort(m_list);
    qDebug() << "冒泡排序执行了" << time.elapsed() << "毫秒";
    emit finish(m_list);
}

void BubbleSort::bubblesort(QVector<int> &list)
{
    int i,j,temp;
    for (j=0;j<list.size()-1;j++)    //用一个嵌套循环来遍历一遍每一对相邻元素 （所以冒泡函数慢嘛，时间复杂度高）
    {
        for (i=0;i<list.size()-1-j;i++)
        {
            if(list[i]>list[i+1])  //从大到小排就把左边的">"改为"<" ！！！
            {
                temp=list[i];      //a[i]与a[i+1](即a[i]后面那个) 交换
                list[i]=list[i+1];    //基本的交换原理"c=a;a=b;b=c"
                list[i+1]=temp;
            }
        }
    }
}

//bubblesort.h
#ifndef BUBBLESORT_H
#define BUBBLESORT_H

#include<QThread>
#include<QVector>
#include<QRandomGenerator>
#include<QElapsedTimer>
#include<QDebug>

class BubbleSort : public QThread
{
    Q_OBJECT
public:
    explicit BubbleSort(QObject *parent = nullptr);

    void recvArray(QVector<int> list);

protected:
    void run() override;

signals:
    void finish(QVector<int> list);
private:
    void bubblesort(QVector<int> &list);
    QVector<int> m_list;
    QRandomGenerator rand;
    QElapsedTimer time;
};

#endif // BUBBLESORT_H

//mainwindow.cpp
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include "generate.h"
#include "bubblesort.h"
#include "quicksort.h"

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    //创建子线程对象
    Generate* gen = new Generate;
    BubbleSort* bubble = new BubbleSort;
    QuickSort* quick = new QuickSort;
    //关联开始信号和子线程接收信号
    connect(this,&MainWindow::starting,gen,&Generate::recvNum);
    //关联按钮点击信号和启动子线程
    connect(ui->start,&QPushButton::clicked,this,[=]()
    {
        emit starting(10000);
        gen->start();
    });
    //关联两个排序线程接收主线程数据
    connect(gen,&Generate::sendArray,bubble,&BubbleSort::recvArray);
    connect(gen,&Generate::sendArray,quick,&QuickSort::recvArray);
    //关联主线程接收排序线程的数据
    connect(bubble,&BubbleSort::finish,this,[=](QVector<int> list)
    {
        for(int i=0;i<list.size();i++)
        {
            ui->bubbleList->addItem(QString::number(list.at(i)));
        }
    });
    connect(quick,&QuickSort::finish,this,[=](QVector<int> list)
    {
        for(int i=0;i<list.size();i++)
        {
            ui->quickList->addItem(QString::number(list.at(i)));
        }
    });
    //关联子线程发射信号和主线程处理函数
    connect(gen,&Generate::sendArray,this,[=](QVector<int>list)
    {
        bubble->start();
        quick->start();
        ui->randList->clear();
        ui->quickList->clear();
        ui->bubbleList->clear();
        for(int i=0;i<list.size();i++)
        {
            ui->randList->addItem(QString::number(list.at(i)));
        }
    });
    //当主窗口关闭时释放所有的内存
    connect(this,&MainWindow::destroyed,this,[=]()
    {
        gen->quit();
        gen->wait();
        gen->deleteLater();

        bubble->quit();
        bubble->wait();
        bubble->deleteLater();

        quick->quit();
        quick->wait();
        quick->deleteLater();
    });

}

MainWindow::~MainWindow()
{
    delete ui;
}

//mainwindow.h
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>

QT_BEGIN_NAMESPACE
namespace Ui { class MainWindow; }
QT_END_NAMESPACE

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();
signals:
    void starting(int num);
private:
    Ui::MainWindow *ui;
};
#endif // MAINWINDOW_H

//generate.cpp
#include "generate.h"

Generate::Generate(QObject *parent)
    : QThread{parent}
{

}

void Generate::recvNum(int num)
{
    m_num = num;
}

void Generate::run()
{
    qDebug() <<"生成随机数的线程地址为：" << QThread::currentThread();
    QVector<int> list;
    time.start();
    for(int i=0;i<m_num;i++)
    {
        list.push_back(rand.generate()%100000);
    }
    qDebug() << "生成" << m_num << "个随机数耗时：" << time.elapsed() << "毫秒";
    emit sendArray(list);
}

```

## 案例九  QObject多线程

要求与案例八相同

```c++
/*仅展示重要部分*/
//bubblesort.cpp
#include "bubblesort.h"

BubbleSort::BubbleSort(QObject *parent)
    : QObject{parent}
{

}

void BubbleSort::working(QVector<int> list)
{
    qDebug()<<"开始排序";
    time.start();
    bubblesort(list);
    qDebug() << "冒泡排序执行了" << time.elapsed() << "毫秒";
    emit finish(list);
}

void BubbleSort::bubblesort(QVector<int> &list)
{
    int i,j,temp;
    for (j=0;j<list.size()-1;j++)    //用一个嵌套循环来遍历一遍每一对相邻元素 （所以冒泡函数慢嘛，时间复杂度高）
    {
        for (i=0;i<list.size()-1-j;i++)
        {
            if(list[i]>list[i+1])  //从大到小排就把左边的">"改为"<" ！！！
            {
                temp=list[i];      //a[i]与a[i+1](即a[i]后面那个) 交换
                list[i]=list[i+1];    //基本的交换原理"c=a;a=b;b=c"
                list[i+1]=temp;
            }
        }
    }
}

//bubblesort.h
#ifndef BUBBLESORT_H
#define BUBBLESORT_H

#include<QObject>
#include<QVector>
#include<QRandomGenerator>
#include<QElapsedTimer>
#include<QDebug>

class BubbleSort : public QObject
{
    Q_OBJECT
public:
    explicit BubbleSort(QObject *parent = nullptr);
    void working(QVector<int> list);
signals:
    void finish(QVector<int> list);
private:
    void bubblesort(QVector<int> &list);
    QRandomGenerator rand;
    QElapsedTimer time;
};

#endif // BUBBLESORT_H

//mainwindow.cpp
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include "generate.h"
#include "bubblesort.h"
#include "quicksort.h"

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    //创建子线程对象
    QThread *t1 = new QThread;
    QThread *t2 = new QThread;
    QThread *t3 = new QThread;
    //创建任务类对象
    Generate* gen = new Generate;
    BubbleSort* bubble = new BubbleSort;
    QuickSort* quick = new QuickSort;
    //将任务类移到子线程中
    gen->moveToThread(t1);
    bubble->moveToThread(t2);
    quick->moveToThread(t3);
    //关联开始信号和子线程接收信号
    connect(this,&MainWindow::starting,gen,&Generate::working);
    //关联按钮点击信号和启动子线程
    connect(ui->start,&QPushButton::clicked,this,[=]()
    {
        t1->start();
        emit starting(10000);
    });
    //关联两个排序线程接收主线程数据
    connect(gen,&Generate::sendArray,bubble,&BubbleSort::working);
    connect(gen,&Generate::sendArray,quick,&QuickSort::working);
    //关联主线程接收排序线程的数据
    connect(bubble,&BubbleSort::finish,this,[=](QVector<int> list)
    {
        for(int i=0;i<list.size();i++)
        {
            ui->bubbleList->addItem(QString::number(list.at(i)));
        }
    });
    connect(quick,&QuickSort::finish,this,[=](QVector<int> list)
    {
        for(int i=0;i<list.size();i++)
        {
            ui->quickList->addItem(QString::number(list.at(i)));
        }
    });
    //关联子线程发射信号和主线程处理函数
    connect(gen,&Generate::sendArray,this,[=](QVector<int>list)
    {
        t2->start();
        t3->start();
        ui->randList->clear();
        ui->quickList->clear();
        ui->bubbleList->clear();
        for(int i=0;i<list.size();i++)
        {
            ui->randList->addItem(QString::number(list.at(i)));
        }
    });
    //当主窗口关闭时释放所有的内存
    connect(this,&MainWindow::destroyed,this,[=]()
    {
        t1->quit();
        t1->wait();
        t1->deleteLater();

        t2->quit();
        t2->wait();
        t2->deleteLater();

        t3->quit();
        t3->wait();
        t3->deleteLater();
    });

}

MainWindow::~MainWindow()
{
    delete ui;
}

//generate.cpp
#include "generate.h"

Generate::Generate(QObject *parent)
    : QObject{parent}
{

}

void Generate::working(int num)
{
    qDebug() <<"生成随机数的线程地址为：" << QThread::currentThread();
    QVector<int> list;
    time.start();
    for(int i=0;i<num;i++)
    {
        list.push_back(rand.generate()%100000);
    }
    qDebug() << "生成" << num << "个随机数耗时：" << time.elapsed() << "毫秒";
    emit sendArray(list);
}

```

## 案例十 QThreadPool多线程

要求与案例八相同

```c++
//bubblesort.cpp
#include "bubblesort.h"

BubbleSort::BubbleSort(QObject *parent)
    : QObject{parent},QRunnable()
{
    setAutoDelete(true);
}

void BubbleSort::recvArray(QVector<int> list)
{
    m_list = list;
}

void BubbleSort::run()
{
    qDebug()<<"开始排序";
    time.start();
    bubblesort(m_list);
    qDebug() << "冒泡排序执行了" << time.elapsed() << "毫秒";
    emit finish(m_list);
}

void BubbleSort::bubblesort(QVector<int> &list)
{
    int i,j,temp;
    for (j=0;j<list.size()-1;j++)    //用一个嵌套循环来遍历一遍每一对相邻元素 （所以冒泡函数慢嘛，时间复杂度高）
    {
        for (i=0;i<list.size()-1-j;i++)
        {
            if(list[i]>list[i+1])  //从大到小排就把左边的">"改为"<" ！！！
            {
                temp=list[i];      //a[i]与a[i+1](即a[i]后面那个) 交换
                list[i]=list[i+1];    //基本的交换原理"c=a;a=b;b=c"
                list[i+1]=temp;
            }
        }
    }
}

//bubblesort.h
#ifndef BUBBLESORT_H
#define BUBBLESORT_H

#include<QObject>
#include<QRunnable>
#include<QVector>
#include<QRandomGenerator>
#include<QElapsedTimer>
#include<QDebug>

class BubbleSort : public QObject , public QRunnable
{
    Q_OBJECT
public:
    explicit BubbleSort(QObject *parent = nullptr);

    void recvArray(QVector<int> list);

protected:
    void run() override;

signals:
    void finish(QVector<int> list);
private:
    void bubblesort(QVector<int> &list);
    QVector<int> m_list;
    QRandomGenerator rand;
    QElapsedTimer time;
};

#endif // BUBBLESORT_H

//mainwindow.cpp
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include "generate.h"
#include "bubblesort.h"
#include "quicksort.h"
#include <QThreadPool>

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    //创建子线程对象
    Generate* gen = new Generate;
    BubbleSort* bubble = new BubbleSort;
    QuickSort* quick = new QuickSort;
    //关联开始信号和子线程接收信号
    connect(this,&MainWindow::starting,gen,&Generate::recvNum);
    //关联按钮点击信号和启动子线程
    connect(ui->start,&QPushButton::clicked,this,[=]()
    {
        emit starting(10000);
        QThreadPool::globalInstance()->start(gen);
    });
    //关联两个排序线程接收主线程数据
    connect(gen,&Generate::sendArray,bubble,&BubbleSort::recvArray);
    connect(gen,&Generate::sendArray,quick,&QuickSort::recvArray);
    //关联主线程接收排序线程的数据
    connect(bubble,&BubbleSort::finish,this,[=](QVector<int> list)
    {
        for(int i=0;i<list.size();i++)
        {
            ui->bubbleList->addItem(QString::number(list.at(i)));
        }
    });
    connect(quick,&QuickSort::finish,this,[=](QVector<int> list)
    {
        for(int i=0;i<list.size();i++)
        {
            ui->quickList->addItem(QString::number(list.at(i)));
        }
    });
    //关联子线程发射信号和主线程处理函数
    connect(gen,&Generate::sendArray,this,[=](QVector<int>list)
    {
        QThreadPool::globalInstance()->start(bubble);
        QThreadPool::globalInstance()->start(quick);
        ui->randList->clear();
        ui->quickList->clear();
        ui->bubbleList->clear();
        for(int i=0;i<list.size();i++)
        {
            ui->randList->addItem(QString::number(list.at(i)));
        }
    });
}

MainWindow::~MainWindow()
{
    delete ui;
}
```

