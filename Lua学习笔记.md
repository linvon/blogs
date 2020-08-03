---
title: "Lua学习笔记"
date: "2018-12-04"
tags: ["Lua"]
categories: ["学习笔记", "lua"]
---
#  基础

- Lua下标从1开始

- 0为true

- 没有continue

- 使用`for key,value in pairs(table) do`遍历数组（table）

- Lua数组的字符串索引问题  

  ```lua
  a = {} 
  x = "y"
  a[x] = 10
  print(a[x]) -- 输出10
  print(a.x)  --输出nil
  print(a.y)  --输出10
  a.x表示以字符串“x”来索引table，a[x]以变量x的值来索引table
  ```
  
- 数组的排序 `tablesort()` 需要被排序数组的索引为`int`类型，否则不能排序，并且需要为索引连续数组

- Lua不能在函数中部分代码段前直接return，即return后必须接end

  ```lua
  function print_r ( t )
  	return -- 会报错
      print(1)
  end
  
  function print_r ( t )
  	if true then 
      	return -- 正确
      end
      print(1)
  end
  ```

- Lua的逻辑运算

  ```lua
  Lua语言中的逻辑运算符包含and,or,not，not和其他高级语言中的！意思是一样的,返回的是一个逻辑值真或者假
  但是and,or和&&,||的区别在于Lua中返回一个运算符两边的值而不是返回一个逻辑值真或者假
  
  a and b -- 当a为真时返回b,当a为假时,返回a    <=> 条件表达式  a?b:a
  a or b  -- 当a为真时返回a, 当a为假时返回b    <=> 条件表达式   a?a:b
  not a   -- 当a为真时返回假,当a为假时返回真    <=> 条件表达式   a?false:true
  ```

- Lua数组判空

  ```lua
  -- Lua中数组判空仅可使用该种方式，其它方式均不可考
  local array = {}
  if next(array) then -- 非空
  	-- do something
  end
  ```



# 技巧

- Lua中功能函数较少，遇到数组或表不能直接打印，因此保存一个打印库可以提高编码效率

  ```lua
  function print_r ( t )
      local print_r_cache={}
      local function sub_print_r(t,indent)
          if (print_r_cache[tostring(t)]) then
              print(indent.."*"..tostring(t))
          else
              print_r_cache[tostring(t)]=true
              if (type(t)=="table") then
                  for pos,val in pairs(t) do
                      if (type(val)=="table") then
                          print(indent.."["..pos.."] => "..tostring(t).." {")
                          sub_print_r(val,indent..string.rep(" ",string.len(pos)+8))
                          print(indent..string.rep(" ",string.len(pos)+6).."}")
                      elseif (type(val)=="string") then
                          print(indent.."["..pos..'] => "'..val..'"')
                      else
                          print(indent.."["..pos.."] => "..tostring(val))
                      end
                  end
              else
                  print(indent..tostring(t))
              end
          end
      end
      if (type(t)=="table") then
          print(tostring(t).." {")
          sub_print_r(t,"  ")
          print("}")
      else
          sub_print_r(t,"  ")
      end
      print()
  end
  ```

- Lua可以直接运行shell命令并获取结果，可以在后台模糊搜索文件并获取整个文件名

  ```lua
  local mod_dir = log_dir .. mod_name   -- log_dir是目的路径 mod_name为模糊搜索关键字
  	getFileTable = io.popen('find ' .. mod_dir)
  	logFileTable = {};
  for file in getFileTable:lines() do
  ```