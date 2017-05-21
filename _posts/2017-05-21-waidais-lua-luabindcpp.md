---
layout: post
title:  "lua绑定cpp对象(闭包与非闭包)"
date:   2017-05-21 22:50:45 +0800
categories: share
---
csdn的账号找不回来了 所以就复制过来
最近在研究lua，自己写了一下lua调用c++对象，思路大概就是用userdata与对象绑定 然后通过元表来实现函数的调用
先随便写个类
头文件:
```
#pragma once
#include<iostream>
class LuaTestObj
{
private:
	int i;
public:
	LuaTestObj(int i);
	~LuaTestObj();
	void sayHello();
	int add(int a, int b);
	int getI();
	void setI(int i);
};
```
cpp:
```
#include "stdafx.h"
#include "LuaTestObj.h"

LuaTestObj::LuaTestObj(int i)
{
	LuaTestObj::i = i;
}

LuaTestObj::~LuaTestObj()
{
}

void LuaTestObj::sayHello()
{
	std::cout << "hello" <<std::endl;
}

int LuaTestObj::add(int a, int b)
{
	return a+b;
}

int LuaTestObj::getI()
{
	return LuaTestObj::i;
}

void LuaTestObj::setI(int i)
{
	LuaTestObj::i = i;
}

```
很简单，就是一个构造函数 一个set一个get 还有个两数相加的add
然后开始实现c++对象与lua的绑定
先写非闭包的实现，大概思路就是lua向cpp传入userdata cpp得到userdata后再来调用指定方法
h:
```
#pragma once
#include "LuaTestObj.h"
extern "C"
{
#include "lua.h"
#include "lualib.h"
#include "lauxlib.h"
}

#define CLASS_NAME "LuaTestObj"
class LuaTestObjWrapper:LuaTestObj
{
public:
	LuaTestObjWrapper(lua_State * L) :LuaTestObj(luaL_checkint(L, 1)) {}
	~LuaTestObjWrapper();
	static void Register(lua_State * L);//注册函数
	static int Constructor(lua_State * L);//构造器

	//下面是要被注册的函数
	static int _sayHello(lua_State * L);
	static int _add(lua_State * L);
	static int _getI(lua_State * L);
	static int _setI(lua_State * L);
};
```
cpp:
```
#include "stdafx.h"
#include "LuaTestObjWrapper.h"


LuaTestObjWrapper::~LuaTestObjWrapper()
{
}

void LuaTestObjWrapper::Register(lua_State * L)
{
	lua_pushcfunction(L, &LuaTestObjWrapper::Constructor);
	lua_setglobal(L, CLASS_NAME);//设置全局的构造函数
}

int LuaTestObjWrapper::Constructor(lua_State * L)
{
	//获取元表 如果元表不存在就新建一个
	if (luaL_newmetatable(L, CLASS_NAME) == 0)
	{
		luaL_getmetatable(L, CLASS_NAME);
	}
	else 
	{
		//这里是用来配置元表的 比如__gc之类的 
		//example:
		/*
		lua_pushstring(L, "__gc");
		lua_pushcfunction(L, &func);
		lua_settable(L, -3);
		*/

		/*-------------------------分割线-------------------------------------------*/

		//这部分是注册函数的
		//新建一个__index表
		lua_newtable(L);//table metatable
		//添加函数
		lua_pushstring(L, "sayHello");//函数名
		lua_pushcfunction(L, &_sayHello);//函数
		lua_settable(L,-3);

		lua_pushstring(L, "add");//函数名
		lua_pushcfunction(L, &_add);//函数
		lua_settable(L, -3);

		lua_pushstring(L, "getI");//函数名
		lua_pushcfunction(L, &_getI);//函数
		lua_settable(L, -3);

		lua_pushstring(L, "setI");//函数名
		lua_pushcfunction(L, &_setI);//函数
		lua_settable(L, -3);
		//绑定到__index
		lua_pushstring(L, "__index");//key table metatable
		lua_insert(L, -2);// table key metatable
		lua_settable(L, -3);//metatable
	}

	auto wrapper = new LuaTestObjWrapper(L);
	auto obj = (LuaTestObjWrapper **)lua_newuserdata(L, sizeof(LuaTestObjWrapper *));//obj metatable
	*obj = wrapper;
	lua_insert(L, -2);//metatable obj
	lua_setmetatable(L, -2);//obj
	return 1;
}

int LuaTestObjWrapper::_sayHello(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	(*obj)->sayHello();
	return 0;
}

int LuaTestObjWrapper::_add(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	int v = (*obj)->add(luaL_checkint(L, 2), luaL_checkint(L, 3));
	lua_pushnumber(L, v);
	return 1;
}

int LuaTestObjWrapper::_getI(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	lua_pushnumber(L, (*obj)->getI());
	return 1;
}

int LuaTestObjWrapper::_setI(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	int i = luaL_checkint(L, 2);
	(*obj)->setI(i);
	return 0;
}
```
下面是闭包的实现:
闭包的话，就是先得到对象与要调用的函数 通过(*obj)->*(func)()来实现,这个时候 就不需要写那么多的静态函数了
h:
```
#pragma once
#include "LuaTestObj.h"
extern "C"
{
#include "lua.h"
#include "lualib.h"
#include "lauxlib.h"
}

#define CLASS_NAME "LuaTestObj"   //用来表示函数名
class LuaTestObjWrapper :LuaTestObj
{
public:
	LuaTestObjWrapper(lua_State * L) :LuaTestObj(luaL_checkint(L, 1)) {}
	~LuaTestObjWrapper();
	static void Register(lua_State * L);//注册函数
	static int Constructor(lua_State * L);//构造器
	static int Closure(lua_State * L);//闭包函数
	//下面注册函数  这里就不需要添加static了
	int _sayHello(lua_State * L);
	int _add(lua_State * L);
	int _getI(lua_State * L);
	int _setI(lua_State * L);
};
```
cpp:
```
#include "stdafx.h"
#include "LuaTestObjWrapper.h"


LuaTestObjWrapper::~LuaTestObjWrapper()
{
}

void LuaTestObjWrapper::Register(lua_State * L)
{
	lua_pushcfunction(L, &LuaTestObjWrapper::Constructor);
	lua_setglobal(L, CLASS_NAME);//设置全局的构造函数
}

int LuaTestObjWrapper::Constructor(lua_State * L)
{
	//获取元表 如果元表不存在就新建一个
	if (luaL_newmetatable(L, CLASS_NAME) == 0)
	{
		luaL_getmetatable(L, CLASS_NAME);
	}
	else 
	{
		//这里是用来配置元表的 比如__gc之类的 
		//example:
		/*
		lua_pushstring(L, "__gc");
		lua_pushcfunction(L, &func);
		lua_settable(L, -3);
		*/

		/*-------------------------分割线-------------------------------------------*/

		//这部分是注册函数的
		//新建一个__index表
		lua_newtable(L);//table metatable
		//添加函数
		lua_pushstring(L, "sayHello");//函数名
		lua_pushcfunction(L, &_sayHello);//函数
		lua_settable(L,-3);

		lua_pushstring(L, "add");//函数名
		lua_pushcfunction(L, &_add);//函数
		lua_settable(L, -3);

		lua_pushstring(L, "getI");//函数名
		lua_pushcfunction(L, &_getI);//函数
		lua_settable(L, -3);

		lua_pushstring(L, "setI");//函数名
		lua_pushcfunction(L, &_setI);//函数
		lua_settable(L, -3);
		//绑定到__index
		lua_pushstring(L, "__index");//key table metatable
		lua_insert(L, -2);// table key metatable
		lua_settable(L, -3);//metatable
	}

	auto wrapper = new LuaTestObjWrapper(L);
	auto obj = (LuaTestObjWrapper **)lua_newuserdata(L, sizeof(LuaTestObjWrapper *));//obj metatable
	*obj = wrapper;
	lua_insert(L, -2);//metatable obj
	lua_setmetatable(L, -2);//obj
	return 1;
}

int LuaTestObjWrapper::_sayHello(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	(*obj)->sayHello();
	return 0;
}

int LuaTestObjWrapper::_add(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	int v = (*obj)->add(luaL_checkint(L, 2), luaL_checkint(L, 3));
	lua_pushnumber(L, v);
	return 1;
}

int LuaTestObjWrapper::_getI(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	lua_pushnumber(L, (*obj)->getI());
	return 1;
}

int LuaTestObjWrapper::_setI(lua_State * L)
{
	auto obj = (LuaTestObjWrapper **)lua_touserdata(L, 1);
	int i = luaL_checkint(L, 2);
	(*obj)->setI(i);
	return 0;
}

```
