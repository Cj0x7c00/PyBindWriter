## PyBindWriter
PyBindWriter is a simple yet effective tool used to bind C++ and python together. 

using the native ctypes library in python, you simply give it a path to you .json file, and
the tool generates a .py file with all the wrappers needed for your DLL

# How To
### C-side 
In your C++ project, you need to create a C API

For the sake of simplicity, were going to define some macros thatll make the API much easier to write

``` C++
#define PY_EXPORT __declspec(dllexport)
#define PY_IMPL_START extern "C" {
#define PY_IMPL_END } // end C linkage
```
#### functions
For functions, you can either implement them directly within the C linkage (as long as its in a .cpp file):
``` C++
// Source.h //
extern "C" {
    PY_EXPORT void InitGLFW();
    PY_EXPORT void Main(const char * name);
}

// Source.cpp //
extern "C" {
    void InitGLFW()
    {
        if (!glfwInit())
        {
            std::cout << "Failed to initialize GLFW\n";
        }
    }

    void Main(const char* name)
    {
        std::cout << "Running...\n";
        GLFWwindow* win = glfwCreateWindow(200, 200, name, NULL, NULL);

        while (!glfwWindowShouldClose(win))
        {
            glfwSwapBuffers(win);
            glfwPollEvents();
        }
    }
}
```
or you can wrap pre-existing ones:
``` C++
void my_func(int a, int b){
    std::cout << a + b << std::endl;
}

PY_IMPL_START
PY_EXPORT void my_func_wrap(int a, int b)
{
    my_func(a, b);
}
PY_IMPL_END
```
### classes


For classes you need to define a ``` PyCreate_YourClassName() ``` for a constructor.

For the deconstructor, youll need to define a ``` PyDestroy_YourClassName(void* self) ```.

- It is very important you add a ``` void* self ``` se that the class gets deleted properly


Write your class:

``` c++

class PY_EXPORT BasicClass
{
public:
	BasicClass();
	~BasicClass();

	void WhatIs(int a, int b);
	int Get();

private:
	int m_i = 2;
};
```

Define a C API to wrap it:
``` C++
// Source.h //
PY_IMPL_START
PY_EXPORT void* PyCreate_BasicClass();
PY_EXPORT void  BasicClass_WhatIs(void* self, int a, int b);
PY_EXPORT int   BasicClass_Get(void* self);
PY_EXPORT void  PyDestroy_BasicClass(void* self);
PY_IMPL_END
```
Note: ```void * self``` is important. As the class instance gets passed into it. then the function is called from the instance.

Impliment the Class and C API 
``` C++
// Source.cpp //
#include "BasicClass.h"
#include <iostream>

BasicClass::BasicClass()
{
	std::cout << "BC says Hello\n";
}

void BasicClass::WhatIs(int a, int b) {
	std::cout << "a + b is: " << a + b << '\n';
}

int BasicClass::Get()
{
	return m_i;
}

BasicClass::~BasicClass()
{
	std::cout << "BC says Goodbye\n";
}
```
Note: ```void * self``` is crucial to defining the member function wrappers. As the library relies on this concept  of passing in an instance then the C API manipulates the instance.
``` C++
PY_IMPL_START
void* PyCreate_BasicClass()
{
	return new BasicClass();
}

void BasicClass_WhatIs(void* self, int a, int b)
{
	reinterpret_cast<BasicClass*>(self)->WhatIs(a, b);
}

int BasicClass_Get(void* self)
{
	return reinterpret_cast<BasicClass*>(self)->Get();
}

void PyDestroy_BasicClass(void* self)
{
	delete reinterpret_cast<BasicClass*>(self);
}
PY_IMPL_END
```


## Binding
1. Create a .json file. name it anything you want.
2. Specify the lib directory, project name, and (optional) a description

``` json
{
    "dllDir": "C:/Users/ex/Desktop/yourProj/Examplecpp/bin/example_cpp.dll",
    "projName": "Example",
    "description": "python Wrapper for the C++ example lib"
}
```

- the ``` "projName" ``` will turn into the name of your python file.

- the ``` "description" ``` will turn into a doc string in the .py file. Its not required, but is recommended.


3. Define your functions and classes
- to define functions:
``` json 
    "funcs":[
        {
            "name": "add",
            "params": ["c_int", "c_int"],
            "return": "c_int"
        }
    ]
```
- to define classes and their member functions:
``` json
  "class": [
    { // Basic Class
      "name": "BasicClass",
      "mfuncs": [
        {
          "name": "PyCreate_BasicClass",
          "params": " ",
          "return": ["c_void_p"]
        },
        {
          "name": "BasicClass_WhatIs",
          "params": ["SELF", "c_int", "c_int"],
          "return": " "
        },
        {
          "name": "PyDestroy_BasicClass",
          "params": ["c_void_p"],
          "return": " "
        }
      ]
    } // End Basic Class
  ]
```
