# 多态实践：继承、覆盖与虚函数

## 测试源码

```c++
#include <iostream>

using namespace std;

class A {
public:
    virtual void print() {
        cout << "in A." << endl;
    }

    virtual void print(int a) {
        cout << "in A-a." << endl;
    }

    void print(double b) {
        cout << "in A-b." << endl;
    }
};

class B : public A {
public:
    void print() {
        cout << "in B." << endl;
    }
};

class C : public B {
public:
    void print() {
        cout << "in C." << endl;
    }

    void print(int a) {
        cout << "in C-a." << endl;
    }

    void print(double b) {
        cout << "in C-b." << endl;
    }
};

int main() {
    A *a1 = new B();
    a1->print();
    a1->print(1);
    a1->print(1.0);

    A *a2 = new C();
    a2->print();
    a2->print(1);
    a2->print(1.0);

    B *b1 = new C();
    b1->print();
    b1->A::print(1);
    b1->A::print(1.0);

    return 0;
}

```

## 运行结果

```text
in B.
in A-a.
in A-b.
in C.
in C-a.
in A-b.
in C.
in A-a.
in A-b.
```

## 注意

- 覆盖针对相同签名函数
- 虚性不会因为非虚签名函数覆盖而消失
- 可见性：同名隐藏
