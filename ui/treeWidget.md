# this sample show how to use tree widget and network and json

MainWidget.h
```cpp
#ifndef MAINWIDGET_H
#define MAINWIDGET_H

#include <QWidget>
#include <QTreeWidget>
#include <QNetworkAccessManager>
#include <QNetworkReply>

namespace Ui {
class MainWidget;
}

class MainWidget : public QWidget
{
    Q_OBJECT

public:
    explicit MainWidget(QWidget *parent = nullptr);
    ~MainWidget();

private slots:
    void fetchTreeList();
    void onFetchTreeListFinished(QNetworkReply* reply);
    void showContextMenu(const QPoint& pos);
    void addGroup();
    void deleteGroup();
    void editGroup();

private:
    Ui::MainWidget *ui;
    QNetworkAccessManager *networkManager;
    QHash<int, QString> groupMap;
    QMenu *contextMenu;
    QAction *addGroupAction;
    QAction *deleteGroupAction;
    QAction *editGroupAction;
    QString configFilePath;
    void loadConfig();
    void saveConfig();
};

#endif // MAINWIDGET_H

```

MainWidget.cpp
```cpp
#include "MainWidget.h"
#include "ui_MainWidget.h"
#include <QMenu>
#include <QAction>
#include <QJsonDocument>
#include <QJsonArray>
#include <QJsonObject>
#include <QFile>
#include <QDir>
#include <QMessageBox>

MainWidget::MainWidget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::MainWidget),
    networkManager(new QNetworkAccessManager(this)),
    configFilePath("config.json")
{
    ui->setupUi(this);
    this->setFixedSize(800, 600);

    connect(networkManager, &QNetworkAccessManager::finished, this, &MainWidget::onFetchTreeListFinished);

    contextMenu = new QMenu(this);
    addGroupAction = contextMenu->addAction("添加组", this, &MainWidget::addGroup);
    deleteGroupAction = contextMenu->addAction("删除组", this, &MainWidget::deleteGroup);
    editGroupAction = contextMenu->addAction("编辑组", this, &MainWidget::editGroup);

    ui->treeWidget->setContextMenuPolicy(Qt::CustomContextMenu);
    connect(ui->treeWidget, &QTreeWidget::customContextMenuRequested, this, &MainWidget::showContextMenu);

    loadConfig();
    fetchTreeList();
}

MainWidget::~MainWidget()
{
    delete ui;
}

void MainWidget::fetchTreeList()
{
    QNetworkRequest request(QUrl("http://127.0.0.1/api/treelist"));
    networkManager->get(request);
}

void MainWidget::onFetchTreeListFinished(QNetworkReply* reply)
{
    if (reply->error() == QNetworkReply::NoError) {
        QByteArray response = reply->readAll();
        QJsonDocument jsonDoc = QJsonDocument::fromJson(response);
        QJsonArray jsonArray = jsonDoc.array();

        QHash<QString, QTreeWidgetItem*> groupItems;

        for (const QJsonValue &value : jsonArray) {
            QJsonObject obj = value.toObject();
            int id = obj["id"].toInt();
            QString name = obj["name"].toString();
            QString group = obj["group"].toString();

            if (group.isEmpty()) {
                group = "未分组";
            }

            if (!groupItems.contains(group)) {
                QTreeWidgetItem *groupItem = new QTreeWidgetItem(ui->treeWidget);
                groupItem->setText(0, group);
                groupItems.insert(group, groupItem);
            }

            QTreeWidgetItem *childItem = new QTreeWidgetItem(groupItems[group]);
            childItem->setText(0, name);

            groupMap.insert(id, group);
        }
    } else {
        QMessageBox::warning(this, "错误", "无法获取数据：" + reply->errorString());
    }
    reply->deleteLater();
}

void MainWidget::showContextMenu(const QPoint &pos)
{
    QTreeWidgetItem *item = ui->treeWidget->itemAt(pos);
    if (item) {
        contextMenu->exec(ui->treeWidget->mapToGlobal(pos));
    }
}

void MainWidget::addGroup()
{
    bool ok;
    QString groupName = QInputDialog::getText(this, "添加组", "组名：", QLineEdit::Normal, QString(), &ok);
    if (ok && !groupName.isEmpty()) {
        QTreeWidgetItem *groupItem = new QTreeWidgetItem(ui->treeWidget);
        groupItem->setText(0, groupName);
        saveConfig();
    }
}

void MainWidget::deleteGroup()
{
    QTreeWidgetItem *item = ui->treeWidget->currentItem();
    if (item && item->parent() == nullptr) {
        delete item;
        saveConfig();
    }
}

void MainWidget::editGroup()
{
    QTreeWidgetItem *item = ui->treeWidget->currentItem();
    if (item && item->parent() == nullptr) {
        bool ok;
        QString groupName = QInputDialog::getText(this, "编辑组", "组名：", QLineEdit::Normal, item->text(0), &ok);
        if (ok && !groupName.isEmpty()) {
            item->setText(0, groupName);
            saveConfig();
        }
    }
}

void MainWidget::loadConfig()
{
    QFile file(configFilePath);
    if (file.open(QIODevice::ReadOnly)) {
        QByteArray data = file.readAll();
        QJsonDocument jsonDoc = QJsonDocument::fromJson(data);
        QJsonObject jsonObject = jsonDoc.object();

        for (const QString &group : jsonObject.keys()) {
            QTreeWidgetItem *groupItem = new QTreeWidgetItem(ui->treeWidget);
            groupItem->setText(0, group);

            QJsonArray children = jsonObject[group].toArray();
            for (const QJsonValue &childValue : children) {
                int id = childValue.toInt();
                if (groupMap.contains(id)) {
                    QTreeWidgetItem *childItem = new QTreeWidgetItem(groupItem);
                    childItem->setText(0, groupMap.value(id));
                }
            }
        }
    }
}

void MainWidget::saveConfig()
{
    QJsonObject jsonObject;
    for (int i = 0; i < ui->treeWidget->topLevelItemCount(); ++i) {
        QTreeWidgetItem *groupItem = ui->treeWidget->topLevelItem(i);
        QString group = groupItem->text(0);

        QJsonArray children;
        for (int j = 0; j < groupItem->childCount(); ++j) {
            QTreeWidgetItem *childItem = groupItem->child(j);
            children.append(groupMap.key(childItem->text(0)));
        }

        jsonObject[group] = children;
    }

    QFile file(configFilePath);
    if (file.open(QIODevice::WriteOnly)) {
        QJsonDocument jsonDoc(jsonObject);
        file.write(jsonDoc.toJson());
    }
}

```

MainWidget.ui
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ui version="4.0">
 <class>MainWidget</class>
 <widget class="QWidget" name="MainWidget">
  <property name="geometry">
   <rect>
    <x>0</x>
    <y>0</y>
    <width>800</width>
    <height>600</height>
   </rect>
  </property>
  <property name="windowTitle">
   <string>MainWidget</string>
  </property>
  <layout class="QHBoxLayout" name="horizontalLayout">
   <item>
    <widget class="QTreeWidget" name="treeWidget">
     <property name="headerHidden">
      <bool>true</bool>
     </property>
     <property name="minimumWidth">
      <number>200</number>
     </property>
    </widget>
   </item>
   <item>
    <spacer name="horizontalSpacer">
     <property name="orientation">
      <enum>Qt::Horizontal</enum>
     </property>
    </spacer>
   </item>
  </layout>
 </widget>
 <resources/>
 <connections/>
</ui>

```

