---
title: "QTcpSocket QAbstractSocket memory leak"
date: 2025-01-04T13:02:41+08:00
draft: false
---

在做一个使用 QTcpSocket 接收大量数据的项目，发现只要是 TcpServer 开始大量发送数据，内存就开始暴涨，使用 Visual Studio 的 HeapProfiling Qt 会分配大量内存，以下是调用栈

```
Qt6Cored.dll!QArrayData::reallocateUnaligned() - Line 244
Qt6Cored.dll!QTypedArrayData<QRingChunk>::reallocateUnaligned() - Line 156
Qt6Cored.dll!QtPrivate::QMovableArrayOps<QRingChunk>::reallocate() - Line 834
Qt6Cored.dll!QArrayDataPointer<QRingChunk>::reallocateAndGrow() - Line 224
Qt6Cored.dll!QArrayDataPointer<QRingChunk>::detachAndGrow() - Line 210
Qt6Cored.dll!QtPrivate::QMovableArrayOps<QRingChunk>::emplace<QRingChunk>() - Line 798
Qt6Cored.dll!QList<QRingChunk>::emplaceBack<QRingChunk>() - Line 887
Qt6Cored.dll!QList<QRingChunk>::append() - Line 469
Qt6Cored.dll!QRingBuffer::reserve() - Line 121
Qt6Networkd.dll!QIODevicePrivate::QRingBufferRef::reserve() - Line 70
Qt6Networkd.dll!QAbstractSocketPrivate::readFromSocket() - Line 1174
Qt6Networkd.dll!QAbstractSocketPrivate::canReadNotification() - Line 626
Qt6Networkd.dll!QAbstractSocketPrivate::readNotification() - Line 39
Qt6Networkd.dll!QAbstractSocketEngine::readNotification() - Line 121
Qt6Networkd.dll!QReadNotifier::event() - Line 1239
Qt6Cored.dll!QCoreApplicationPrivate::notify_helper() - Line 1342
Qt6Cored.dll!doNotify() - Line 1269
Qt6Cored.dll!QCoreApplication::notify() - Line 1253
Qt6Cored.dll!QCoreApplication::notifyInternal2() - Line 1168
Qt6Cored.dll!QCoreApplication::sendEvent() - Line 1613
Qt6Cored.dll!qt_internal_proc() - Line 157
user32.dll!0x7ffac3fa5801()
user32.dll!0x7ffac3fa334d()
Qt6Cored.dll!QEventDispatcherWin32::processEvents() - Line 539
Qt6Cored.dll!QEventLoop::processEvents() - Line 104
Qt6Cored.dll!QEventLoop::exec() - Line 194
Qt6Cored.dll!QCoreApplication::exec() - Line 1513
untitled15.exe!main() - Line 21
untitled15.exe!invoke_main() - Line 79
untitled15.exe!__scrt_common_main_seh() - Line 288
untitled15.exe!__scrt_common_main() - Line 331
untitled15.exe!mainCRTStartup() - Line 17
kernel32.dll!0x7ffac418e8d7()

```

观察到的问题，可以用以下的小程序来复现，复现的条件是 TcpServer 要大量的发送数据
```c++
#include <QCoreApplication>

#include <QDebug>
#include <QTcpSocket>

int main(int argc, char* argv[])
{
	QCoreApplication a(argc, argv);

	auto s = new QTcpSocket(0);
	s->connectToHost("127.0.0.1", 17725);
	s->waitForConnected(-1);
	int i = 0;

	QObject::connect(s, &QTcpSocket::readyRead, [&]() {
		s->read(12);
		qDebug() << i;
		return 0;
		});

	return a.exec();
}
```

如果把 QTcpSocket 的 read 函数当作普通的 read 函数，就无法解释这个问题，因为即使对端发送了大量的数据，这边即使读取的慢也不应该引起内存泄露。但是问题是，这不是普通的 file descriptor，QTcpSocket(QAbstractSocket) 会在 eventloop 中做一些工作，包括读取 file descriptor 并把它存放在 QAbstractSocket 的一个 buffer 中，如果在 readyRead 中只读取了少量的数据，这样 QAbstractSocket buffer 中的数据就会累积，导致内存暴涨。 解决方法也简单，在收到 readyRead signals 时，一次把所有的数据都读取出来，

```c
		while (s->bytesAvailable() > 0) {
			s->read(12);
		}
```

Reference:

> The readyRead() signal is emitted every time a new chunk of data has arrived. bytesAvailable() then returns the number of bytes that are available for reading. Typically, you would connect the readyRead() signal to a slot and read all available data there. If you don't read all the data at once, the remaining data will still be available later, and any new incoming data will be appended to QAbstractSocket's internal read buffer. To limit the size of the read buffer, call setReadBufferSize().