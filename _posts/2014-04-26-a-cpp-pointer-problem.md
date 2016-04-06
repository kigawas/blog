---
title:      一个C++多级指针问题
excerpt: 大一下学期时候的例题
---
```cpp
//请问下面的代码输出结果是？
//答案：You learn C++ language very well!

#include <iostream>
#include <cstring>
using namespace std;

int main()
{
    char *c [ ] = { "John learn C++ language", "Be well!", "You", "Not very" };

    char **p [ ] = { c+3, c+2, c+1, c }; //这个数组的元素是指向字符串的指针
    //指针指向的字符串内容分别为：
    //c[3]: "Not very"
    //c[2]: "You"
    //c[1]: "Be well!"
    //c[0]: "John learn C++ language"

    char ***pp  = p;
    //该指针指向p数组的首元素，也即是字符串"Not very"的首字符地址

    cout << ( **++pp );
    //此时pp指向p [ 1 ]，进行一次解引用并输出p [ 1 ]，其内容为"You"

    cout << ( *--*++pp + 4 );
    //此时pp指向p [ 2 ]，对pp进行一次解引用，访问p[ 2 ]并将p[ 2 ]的值改为c，
    //此时p数组为 p [ ] ={  c+3, c+2, c, c }，再进行一次解引用，从p [ 2 ] 访问c[ 0 ]
    //从c[ 0 ]首字符地址后4字节地址开始，输出其后的字符串
    //其内容为" learn C++ language"

    cout << ( *pp [ -2 ] + 3 );
    // *pp [ - 2 ] <=> * ( * ( pp - 2 )  )  此时pp指向p [ 2 ]，
    //从p [ 0 ]后3字节地址开始输出其后字符直到'\0' ,字符串内容为" very"

    cout << ( pp [ -1 ] [ -1 ] + 2 );
    //pp [ -1 ] [ -1 ] <=> * ( ( * ( pp - 1 )  )  - 1 )此时pp指向p [ 2 ],
    //依次访问 p [ 1 ]和c [ 1 ]，从c [ 1 ]地址后3字节开始输出字符串，内容为" well!"

    cout << endl;
    return 0;
}
```

注意：
在C/C++中，形如`a[1]`访问数组元素的syntax是一个[语法糖][]，其含义是`*( a + 1 )`，如果将`a[1]`换成`1[a]`也是等价的。

[语法糖]:http://zh.wikipedia.org/wiki/%E8%AF%AD%E6%B3%95%E7%B3%96
