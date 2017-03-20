---
layout: post
title: 分别用 Scheme 和 C 实现的十六进制查看器，以及异或加密工具
date: 2014-06-19
---

先放用 C 写的版本。

一、十六进制查看器：

```c
/******************************
 * 十六进制文件查看器
 * ****************************/
#include <stdio.h>

int main(int argc, char *argv[])
{
    char* fname;
    fname = argv[1];
    FILE *fp;
    fp = fopen(fname, "rb");

    if (fp != NULL)
    {
        int i, b, puted, blank, count = 1;
        int temp[16];

        printf("00000000  "); //地址头，硬编码
        while (! feof(fp)) {
            b = fgetc(fp);
            if (b != EOF) {
                printf("%02x ", b); //输出字节的16进制数
                temp[count % 16 - 1] = b;
            }
            if (count % 16 == 0) {
                printf(" |");
                for (i = 0; i < 16; i++) {
                    if (temp[i] > 31 && temp[i] < 127)
                        putchar(temp[i]);  //试图以ASCII输出
                    else
                        putchar('.');      //不可打印字符用 . 代替
                }             
                printf("|\n%08x  ", count);    //将计数器作为地址输出            
            }
            else if (count % 8 == 0) {  //输出8字节后，加一个空格分列
                printf(" ");
            }
            
            count++;
        }

        //如果最后一行不足16个字节，用空格补足
        if ((count - 2) % 16 != 0) {  //count在最后一次循环后++了，所以这里要减去1
            puted = (count - 2) % 16; //但是我发现要减去2才对，为什么呢？
            if (puted > 8) {          //最后一行输出少于8个字节的，多加一个空格
                blank = (16 - puted) * 3 - 1;
            } else {
                blank = (16 - puted) * 3;
            }
            for (i = 0; i < blank; i++) {
                printf(" ");
            }
            printf("  |");  //把临时数组中剩余的数打印出来
            for (i = 0; i < puted; i++) {
                if (temp[i] > 32 && temp[i] < 127) {
                    putchar(temp[i]);
                }
                else {
                    putchar('.');
                }
            }
            for (i = 0; i < 16 - puted; i++) {  
                printf(" ");  //输出最后的空格，撑满 |.....|
            }
            printf("|\n");
        }
    }
    else {
        printf("Fail to open file %s!\n", fname);
    }

    fclose(fp);
    
    return 0;
}
```

本来代码很短，之所以搞这么乱是因为我想模仿 hexdump 的输出。

二、异或加密

```c
#include <stdio.h>
#define PASSWD 21

int main(int argc, char *argv[])
{
    char* in_file;
    char* out_file;
    in_file = argv[1];
    out_file = argv[2];
    
    FILE *fpi, *fpo;
    fpi = fopen(in_file, "rb");
    fpo = fopen(out_file, "wb");

    if (fpi != NULL)
    {
        while (! feof(fpi)) {
            int b = fgetc(fpi);
            if (b != EOF) {
                fputc(b ^ PASSWD, fpo);
            }
        }
    }
    else {
        printf("Fail to open file %s!\n", in_file);
    }

    fclose(fpi);
    fclose(fpo);

    return 0;
}
```

三、用 Scheme 写的十六进制查看器

```scheme
#!/usr/local/bin/csi -s

(define num->hex
  (lambda (n)
    (if (< n 16)
        (string-append "0" (number->string n 16))
        (number->string n 16))))

(define addr
  (lambda (n)
    (cond
     ((< n 256)
      (string-append "000000" (num->hex n)))
     ((< n #x1000)
      (string-append "00000" (num->hex n)))
     ((< n #x10000)
      (string-append "0000" (num->hex n)))
     ((< n #x100000)
      (string-append "000" (num->hex n)))
     ((< n #x1000000)
      (string-append "00" (num->hex n)))
     ((< n #x10000000)
      (string-append "0" (num->hex n)))
     (else (num->hex n)))))

(define file->list
  (lambda (file)
    (call-with-input-file file
      (lambda (port)
        (let iter ((b (read-byte port)))
          (if (eof-object? b) '()
              (cons b (iter (read-byte port)))))))))

(define bin
  (lambda (file)
    (let iter ((l (file->list file))
               (count 1))
      (if (not (null? l))
          (begin
            (if (= count 1)
                (display "00000000  ")
                (display ""))
            (display (num->hex (car l))) (display " ")
            (cond ((= (modulo count 16) 0)
                   (begin
                     (newline)
                     (display (addr count))
                     (display "  ")))
                  ((= (modulo count 8) 0)
                   (display " ")
                   (display "")))
            (iter (cdr l) (+ count 1)))
          (newline)))))

(let ((f (car (command-line-arguments))))
  (bin f))
```

四、Scheme 版的加密工具

```scheme
#!/usr/local/bin/csi -s

(define encrypt
  (lambda (n)
    (bitwise-xor n 21)))

(define file-read-loop
  (lambda (file)
    (call-with-input-file file
      (lambda (port)
        (let iter ((b (read-byte port)))
          (if (eof-object? b) '()
              (cons (encrypt b)
                    (iter (read-byte port)))))))))

(define file-write-loop
  (lambda (outf bl)
    (call-with-output-file outf
      (lambda (port)
        (let iter ((l bl))
          (if (null? l) -1
              (begin
                (write-byte (car l) port)
                (iter (cdr l)))))))))

(let* ((infile
        (car (command-line-arguments)))
       (outfile
        (string-append infile ".enc")))
  (file-write-loop outfile (file-read-loop infile)))
```

Scheme 的输入和输出果断是硬伤，标准库太弱，每个实现五花八门。