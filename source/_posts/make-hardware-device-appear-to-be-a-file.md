title: Linux是如何将硬件设备呈现为文件的
categories: Linux
date: 2016-11-09 16:23:10
tags: Linux
description: Linux是如何将硬件设备呈现为文件的
---

## 前言

Linux给我们最大的错觉就是一切皆文件，那么它是如何将硬件设备也呈现为文件的呢？

## 背景

- 要理解Linux驱动程序设计，需要了解正常的应用程序如何获得对操作系统提供的服务的访问
- 应用程序是间接访问的——通过特别的受保护的接口（比如系统调用）——通常由库函数来实现

## 标准的文件I/O方法

- int open( char *pathname, int flags, … );
- int read( int fd, void *buf, size_t count );
- int write( int fd, void *buf, size_t count );
- int lseek( int fd, loff_t offset, int whence ); 
- int close( int fd );
- 我们可以使用`$ man 2 <方法名>`来查看用法说明。

## 一个标准文件I/O方法用例

```
#include <fcntl.h>	// for open() 
#include <stdio.h>	// for printf(), perror() 
#include <unistd.h>	// for read(), close() 
#include <string.h>	// for strncmp()

char buf[4];

int main( int argc, char **argv )
{
	if ( argc == 1 ) return -1;

	int	fd = open( argv[1], O_RDONLY );
	if ( fd < 0 ) { perror( argv[1] ); return -1; }

	int nbytes = read( fd, buf, 4 );
	if ( nbytes < 0 ) { perror( argv[1] ); return -1; }

	if ( strncmp( buf, "\177ELF", 4 ) == 0 )
		printf( "File \'%s\' has an ELF signature \n", argv[1] );
	else	printf( "File \'%s\' is not an ELF file \n", argv[1] );
	close( fd );
}
```

## 特殊的“设备”文件

- Unix系统把硬件设备看做特殊的文件，这样的话编程人员就是用上面那些熟悉的方法来访问设备了
- 系统管理员必须要创建这些设备文件（在/dev目录下）
- 编写设备驱动

<!-- more -->

## RTC例子

- 在上一篇文章中我们学习了Real-Time-Clock设备可以通过使用inb()和outb()的模块进行访问。

```
outb( addr, 0x70 );
outb( data, 0x71 );

outb( addr, 0x70 );
data = inb( 0x71 );
```
- 所以我们可以创建一个简单的字符设备将RTC内存看做是在一个普通的文件中。

## 如何访问设备

![设备访问工作原理](https://raw.githubusercontent.com/rason/rason.github.io/master/image/device-access-works.png)

## cmosram.c驱动

```
#include <linux/module.h>	// for printk() 
#include <linux/fs.h>		// for register_chrdev() 
#include <asm/uaccess.h>	// for put_user(), get_user()
#include <asm/io.h>		// for inb(), outb()

char modname[] = "cmosram";	// name of this kernel module
char devname[] = "cmos";	// name for the device's file
int	my_major = 70;		// major ID-number for driver
int	cmos_size = 128;	// total bytes of cmos memory
int	write_max = 9;		// largest 'writable' address

ssize_t my_read( struct file *file, char *buf, size_t len, loff_t *pos )
{
	unsigned char	datum;

	if ( *pos >= cmos_size ) return 0;

	outb( *pos, 0x70 );  datum = inb( 0x71 );

	if ( put_user( datum, buf ) ) return -EFAULT;

	*pos += 1;
	return	1;
}

ssize_t my_write( struct file *file, const char *buf, size_t len, loff_t *pos )
{
	unsigned char	datum;

	if ( *pos >= cmos_size ) return 0;

	if ( *pos > write_max ) return -EPERM;

	if ( get_user( datum, buf ) ) return -EFAULT;

	outb( *pos, 0x70 );  outb( datum, 0x71 );

	*pos += 1;
	return	1;
}

loff_t my_llseek( struct file *file, loff_t pos, int whence )
{
	loff_t	newpos = -1;

	switch ( whence )
		{
		case 0:	newpos = pos; break;			// SEEK_SET
		case 1: newpos = file->f_pos + pos; break;	// SEEK_CUR
		case 2: newpos = cmos_size + pos; break;	// SEEK_END
		}

	if (( newpos < 0 )||( newpos > cmos_size )) return -EINVAL;
	
	file->f_pos = newpos;
	return	newpos;
}


struct file_operations my_fops = {
				owner:	THIS_MODULE,
				llseek:	my_llseek,
				write:	my_write,
				read:	my_read,
				};

static int __init my_init( void )
{
	printk( "<1>\nInstalling \'%s\' module ", devname );
	printk( "(major=%d) \n", my_major );
	return	register_chrdev( my_major, devname, &my_fops );
}

static void __exit my_exit(void )
{
	unregister_chrdev( my_major, devname );
	printk( "<1>Removing \'%s\' module\n", devname );
}

module_init( my_init );
module_exit( my_exit );
MODULE_LICENSE("GPL"); 

```

- 编译：`$ make`
- 安装：`$ sudo /sbin/insmod cmosram.ko`
- 创建设备文件：`$ sudo mknod  /dev/cmos  c  70  0`

## 查看设备文件中的内容

- 上面我们已经创建了设备文件
- 还需要一个工具类来查看设备文件的内容

```
dump.cpp

#include <stdio.h>	// for printf(), fprintf(), perror()
#include <fcntl.h>	// for open(), O_RDONLY
#include <unistd.h>	// for lseek(), read(), close()
#include <stdlib.h>	// for exit()

int main( int argc, char *argv[] )
{
	// check for mandatory command-line argument
	if ( argc == 1 ) 
		{
		fprintf( stderr, "You must specify an input-file\n" );
		exit(1);
		}
	
	// open the specified input-file and obtain its size (in bytes)
	int	fd = open( argv[1], O_RDONLY );
	if ( fd < 0 ) { perror( argv[1] ); exit(1); }
	int	filesize = lseek( fd, 0, SEEK_END );
	
	// display title for the screen output (file's name and size)
	printf( "\nFILE DUMP: %s (%d bytes)\n", argv[1], filesize );
	
	// main loop to display file contents in hex and ascii formats
	lseek( fd, 0, SEEK_SET );
	for (int i = 0; i < filesize; i += 16)
		{
		printf( "\n  %08X: ", i );
		unsigned char	buf[ 16 ];
		for (int j = 0; j < 16; j++) 
			if ( read( fd, buf+j, 1 ) < 0 ) break;
		for (int j = 0; j < 16; j++) 
			{
			if ( i+j < filesize ) printf( "%02X ", buf[j] );
			else	printf( "   " );
			}
		printf( " " );
		for (int j = 0; j < 16; j++)
			{
			unsigned char	ch = (i+j < filesize) ? buf[j] : ' ';
			if (( ch < 0x20 )||( ch > 0x7E )) ch = '.';	
			printf( "%c", ch );
			}	
		}
	close( fd );
	printf( "\n\n" );
}
```

- 编译：`$ g++ dump.cpp -o dump`
- 用法：`$ ./dump /dev/cmos`
- 可以看到CMOS内存中的128个字节中的内容如下图所示

![CMOS内存中的内容](https://raw.githubusercontent.com/rason/rason.github.io/master/image/cmos-memery.png)

对应上篇博客中每个字节表示的内容，可以看出系统的硬件时间。

## 总结

所以，Linux其实就是访问设备驱动模块，设备驱动对硬件进行读写，这样就看起来是对文件进行读写了。
