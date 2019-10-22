# 指针与引用



```cpp
#include <iostream>

using namespace std;

void print(int num){
    num=1;
    cout<< "print " << num <<endl;
    cout<< "print location " << &num <<endl;
}

void printPoint(int *num){
    int tmp =2;
    num=&tmp;
    cout<< "printPoint " << *num <<endl;
    cout<< "printPoint location " << num <<endl;
}

void printReference(int &num){
    int tmp = 3;
    num = tmp;
    cout<< "printReference " << num <<endl;
    cout<< "printReference location " << &num <<endl;
}

int main(){
    int a = 0;
    print(a);
    cout<< "after print " << a <<endl;
    cout<< "after print location " << &a <<endl;
    printPoint(&a);
    cout<< "after printPoint " << a <<endl;
    cout<< "after printPoint location " << &a <<endl;
    printReference(a);
    cout<< "after printReference " << a <<endl;
    cout<< "after printReference location " << &a <<endl;
}

/* output */
/* print 1 */
/* print location 0x7fffbb625d7c */
/* after print 0 */
/* after print location 0x7fffbb625dac */
/* printPoint 2 */
/* printPoint location 0x7fffbb625d74 */
/* after printPoint 0 */
/* after printPoint location 0x7fffbb625dac */
/* printReference 3 */
/* printReference location 0x7fffbb625dac */
/* after printReference 3 */
/* after printReference location 0x7fffbb625dac */
```

>  https://blog.csdn.net/luoshenfu001/article/details/8601494 