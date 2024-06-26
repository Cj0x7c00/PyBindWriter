# Getting started

|Table of contents|
|---------------|
|[1. Introduction](#1-introduction)|
|[2. C/C++ Side of things](#c-side)|
|[3. Compile](#compiling)          |
|[4. Binding](#binding)            |
|[Optional Flags](#flags)          |
|[5. Final Step](#final-step)      |



## C-side 
In your C++ project, you need to create a C API

For the sake of simplicity, were going to define some macros thatll make the API much easier to write

``` C++
#define PY_EXPORT __declspec(dllexport)
#define PY_IMPL_START extern "C" {
#define PY_IMPL_END } // end C linkage
```
### functions
For functions, you can either implement them directly within the C linkage (as long as its in a .cpp file):
``` C++
// Source.h //
PY_IMPL_START
PY_EXPORT void InitGLFW();
PY_EXPORT void Main(const char * name);
PY_IMPL_END

// Source.cpp //
PY_IMPL_START
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
PY_IMPL_END
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

class BasicClass
{
public:
	BasicClass();
	~BasicClass();

	void WhatIs(int a, int b);
};
```

Define a C API to wrap it:
``` C++
// Source.h //
PY_IMPL_START
PY_EXPORT void* PyCreate_BasicClass();
PY_EXPORT void  BasicClass_WhatIs(void* self, int a, int b);
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

void PyDestroy_BasicClass(void* self)
{
	delete reinterpret_cast<BasicClass*>(self);
}
PY_IMPL_END
```

## Compiling
Compile your C/C++ program into a .DLL/.SO.


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
            "name": "Main",
            "params": ["c_char_p"],
            "return": " "
        }
    ]
```
- to define classes and their member functions:
``` json
  "class": [
    { 
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
    } 
  ]
```

## Flags
|Flag|Usage|
|---|---|
|`-v`| verbose   |
|`-o`| output dir|
|`-h`| help      |

## Final Step
- Run the script using python.
  - pass in the path to your .json file
  - It will generate a .py file with the same name specified in `"projName"`

*Example*
``` shell
python3 BindWriter.py "path/to/json/file.json" <flags>
```

The .py file will contain the necessary boilerplate code for your python wrapper. And youll be ready to use your library in python
