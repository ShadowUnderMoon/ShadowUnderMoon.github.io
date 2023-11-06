# simple unique_ptr implementation

```c++
#include <cassert>
#include <iostream>
#include <fmt/core.h>
#include <fmt/ranges.h>
#include <fmt/format.h>
#include <fmt/os.h>
#include <vector>
#include <array>
#include <fstream>
using namespace std;

struct B
{
    virtual ~B() = default;
    virtual void bar() {std::cout << "B::bar\n"; }
};

struct D: B
{
    D() {std::cout << "D::D\n"; }
    ~D() { std::cout << "D::~D\n"; }
    void bar() override { std::cout << "D::bar\n"; }
};


void close_file(FILE* fp)
{
    cout << "close file\n";
    fclose(fp);
}

template<class T>
class Deleter
{
public:
    void operator() (T* p) const
    {
        delete p;
    }

};

template<class T, class Deleter=Deleter<T>>
class Unique_ptr
{
private:
    T* m_p;
    Deleter m_deleter;
public:
    template<class U, class E>
    friend class Unique_ptr;
    Unique_ptr(T* p) {
        m_p = p;
    }
    Unique_ptr(T* p, Deleter deleter) {
        m_p = p;
        m_deleter = deleter;
    }
    Unique_ptr(Unique_ptr&& u) {
        cout << __PRETTY_FUNCTION__ << endl;
        m_p = u.m_p;
        u.m_p = nullptr;
    }
    template<class U, class E>
    Unique_ptr(Unique_ptr<U, E>&& u) {
        cout << __PRETTY_FUNCTION__ << endl;
        m_p = u.m_p;
        u.m_p = nullptr;
        // m_deleter = u.m_deleter;
    }
    T* get() const {
        return m_p;
    }
    Deleter get_deleter() const {
        return m_deleter;
    }
    operator bool() const {
        return bool(m_p);
    }
    T* operator ->() const {
        return m_p;
    }
    ~Unique_ptr() {
        m_deleter(m_p);
    }
};

template<class T, class Deleter=Deleter<T>>
Unique_ptr<T, Deleter> make_Unique() {
    return Unique_ptr<T, Deleter>(new T);
}


Unique_ptr<D> pass_through(Unique_ptr<D> p)
{
    p->bar();
    return p;
}
int main()
{
    cout << "1) Unique ownership sematics demo\n";
    {
        Unique_ptr<D> p = make_Unique<D>();
        Unique_ptr<D> q = pass_through(std::move(p));
        assert(!p);
    }

    cout << "\n" "2) Runtime polymorphism demo\n";
    {
        Unique_ptr<B> p = make_Unique<D>();
        p->bar();
    }

    cout << "\n" "3) Custom deleter demo\n";
    ofstream("demo.txt") << "x";
    {
        using Unique_file_t = Unique_ptr<FILE, decltype(&close_file)>;
        Unique_file_t fp(fopen("demo.txt", "r"), &close_file);
        if (fp)
            cout << char(fgetc(fp.get())) << "\n";
    }

    cout << "\n" "4) Custom lambda-expression deleter and exception safety demo\n";
    try
    {
        Unique_ptr<D, void(*)(D*)> p(new D, [](D* ptr) 
        {
            cout << "destroying from a custom deleter...\n";
            delete ptr;
        });
        throw runtime_error("");
    }
    catch (const std::exception&)
    {
        cout << "Caught exception\n";
    }

}
```
