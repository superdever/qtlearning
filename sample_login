# Login Widget sample

LoginWidget.h
```cpp
#ifndef LOGINWIDGET_H
#define LOGINWIDGET_H

#include <QWidget>

namespace Ui {
    class LoginWidget;
}

class LoginWidget : public QWidget
{
    Q_OBJECT

public:
    explicit LoginWidget(QWidget* parent = nullptr);
    ~LoginWidget();

signals:
    void loginSuccessful();

private slots:
    void on_loginButton_clicked();
    void on_cancelButton_clicked();

private:
    Ui::LoginWidget* ui;
};

#endif // LOGINWIDGET_H

```
LoginWidget.cpp
```cpp
#include "LoginWidget.h"
#include "ui_LoginWidget.h"
#include <QMessageBox>

LoginWidget::LoginWidget(QWidget* parent) :
    QWidget(parent),
    ui(new Ui::LoginWidget)
{
    ui->setupUi(this);
    this->setFixedSize(800, 600);
    this->setStyleSheet("background-color: green; color: black;");
}

LoginWidget::~LoginWidget()
{
    delete ui;
}

void LoginWidget::on_loginButton_clicked()
{
    QString username = ui->usernameLineEdit->text();
    QString password = ui->passwordLineEdit->text();

    // 简单验证逻辑
    if (username == "user" && password == "password") {
        emit loginSuccessful();
    }
    else {
        QMessageBox::warning(this, "登录失败", "用户名或密码错误！");
    }
}

void LoginWidget::on_cancelButton_clicked()
{
    QApplication::quit();
}

```

LoginWidget.ui
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ui version="4.0">
	<class>LoginWidget</class>
	<widget class="QWidget" name="LoginWidget">
		<property name="geometry">
			<rect>
				<x>0</x>
				<y>0</y>
				<width>800</width>
				<height>600</height>
			</rect>
		</property>
		<property name="windowTitle">
			<string>LoginWidget</string>
		</property>
		<property name="styleSheet">
			<string notr="true">background-color: green; color: black;</string>
		</property>
		<layout class="QVBoxLayout" name="verticalLayout">
			<item>
				<spacer name="verticalSpacer">
					<property name="orientation">
						<enum>Qt::Vertical</enum>
					</property>
				</spacer>
			</item>
			<item>
				<layout class="QHBoxLayout" name="horizontalLayout">
					<item>
						<spacer name="horizontalSpacer">
							<property name="orientation">
								<enum>Qt::Horizontal</enum>
							</property>
						</spacer>
					</item>
					<item>
						<widget class="QGroupBox" name="loginGroupBox">
							<property name="title">
								<string>登录</string>
							</property>
							<property name="maximumSize">
								<size>
									<width>200</width>
									<height>16777215</height>
								</size>
							</property>
							<layout class="QFormLayout" name="formLayout">
								<item row="0" column="0">
									<widget class="QLabel" name="usernameLabel">
										<property name="text">
											<string>用户名：</string>
										</property>
									</widget>
								</item>
								<item row="0" column="1">
									<widget class="QLineEdit" name="usernameLineEdit"/>
								</item>
								<item row="1" column="0">
									<widget class="QLabel" name="passwordLabel">
										<property name="text">
											<string>密码：</string>
										</property>
									</widget>
								</item>
								<item row="1" column="1">
									<widget class="QLineEdit" name="passwordLineEdit">
										<property name="echoMode">
											<enum>QLineEdit::Password</enum>
										</property>
									</widget>
								</item>
								<item row="2" column="0" colspan="2">
									<widget class="QCheckBox" name="rememberPasswordCheckBox">
										<property name="text">
											<string>记住密码</string>
										</property>
									</widget>
								</item>
								<item row="3" column="0" colspan="2">
									<widget class="QCheckBox" name="autoLoginCheckBox">
										<property name="text">
											<string>自动登录</string>
										</property>
									</widget>
								</item>
								<item row="4" column="0">
									<widget class="QPushButton" name="loginButton">
										<property name="text">
											<string>登录</string>
										</property>
									</widget>
								</item>
								<item row="4" column="1">
									<widget class="QPushButton" name="cancelButton">
										<property name="text">
											<string>取消</string>
										</property>
									</widget>
								</item>
							</layout>
						</widget>
					</item>
					<item>
						<spacer name="horizontalSpacer_2">
							<property name="orientation">
								<enum>Qt::Horizontal</enum>
							</property>
						</spacer>
					</item>
				</layout>
			</item>
			<item>
				<spacer name="verticalSpacer_2">
					<property name="orientation">
						<enum>Qt::Vertical</enum>
					</property>
				</spacer>
			</item>
		</layout>
	</widget>
	<resources/>
	<connections/>
</ui>

```
