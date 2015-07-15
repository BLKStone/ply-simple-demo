title: PLY(Python Lex Yacc)简明笔记
categories: python
tags:
  - python
  - 编译原理
description: python lex yacc usage simple demos.
date: 2015-07-15 20:01:50
---
#说明
本笔记是将 Lex&YACC HOWTO\[[1](http://segmentfault.com/a/1190000000396608#articleHeader24)\] 的部分内容用PLY重新实现。
本文不是正则表达式的教程，相关内容请寻找其他教程。
PLY文档\[[2](http://www.pchou.info/open-source/2014/01/18/52da47204d4cb.html)\]的翻译(以及原文)也是非常重要的参考资料。

#安装PLY

> pip install ply

#lex简易示例
##认识lex

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-  
# exam1.py

import ply.lex as lex

tokens = ['START','STOP']

def t_START(t):
    r'start'
    print "start command received"
    return t

def t_STOP(t):
    r'stop'
    print "stop command received"
    return t

# 行号统计
def t_newline(t):
    r'\n+'
    t.lexer.lineno += t.value.count("\n")

# 出错处理
def t_error(t):
    print "Illegal character '%s'" % t.value[0]
    t.lexer.skip(1)


# Build the lexer
lexer = lex.lex()

# 测试数据
s = '''
stop and start
'''

# Give the lexer some input
lexer.input(s)

while True:
    tok = lexer.token()
    if not tok: break

```

##一个更复杂的示例
假设下面是一个我们想解析的文件：
```
logging {
    category lame−servers { null; };
    category cname { null; };
};

zone "." {
    type hint;
    file "/etc/bind/db.root";
};
```

这个文件中有以下几类符号(*tokens*)


1. WORDs ，如*zone*和*type*

2. FILENAMEs ，如*/etc/bind/db.root*

3. QUOTEs ，如包括文件名的符号

4. OBRACEs ，左花括号*{*

5. EBRACEs ，右花括号*}*

6. SEMICOLONs ，*;*

对应的lex文件如下

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
# exam2.py  

import ply.lex as lex

tokens = ["WORD","FILENAME","QUOTE","OBRACE","EBRACE","SEMICOLON"]

def t_WORD(t):
    r'[a-zA-Z][a-zA-Z0-9-]*'
    print "WORD ",
    return t

def t_FILENAME(t):
    r'[a-zA-Z0-9/.-]+'
    print "FILENAME ",
    return t

def t_QUOTE(t):
    r'"'
    print "QUOTE ",
    return t

def t_OBRACE(t):
    r'{'
    print "OBRACE ",
    return t

def t_EBRACE(t):
    r'}'
    print "EBRACE ",
    return t

def t_SEMICOLON(t):
    r';'
    print "SEMICOLON ",
    return t


# 不做处理的符号 空格与tab
t_ignore = " \t"

# 行号统计
def t_newline(t):
    r'\n+'
    t.lexer.lineno += t.value.count("\n")
    print ''

# 出错处理
def t_error(t):
    print "Illegal character '%s'" % t.value[0]
    t.lexer.skip(1)

# Build the lexer
lexer = lex.lex()


file_object = open('test.conf')
s = file_object.read()
print s


# Give the lexer some input
lexer.input(s)

while True:
    tok = lexer.token()
    if not tok: break
```

#yacc示例
##一个简单的温度调节控制器
我们想用一门简单的语言去控制一个温度调节器，例如：
```
heat on
    Heater on!
heat off
    Heater off!
target temperature 22
    New temperature set!
```
我们需要辨别的符号有：*heat*,on/off(*STATE*),*target*,*temperature*,*NUMBER*。对应的lex文件如下（Example 3）:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-  
# exam3lex.py

import ply.lex as lex

tokens = ['NUMBER','TOKHEAT','STATE','TOKTARGET','TOKTEMPRATURE']

def t_NUMBER(t):
    r'[0-9]+'
    return t;

def t_TOKHEAT(t):
    r'heat'
    return t

def t_STATE(t):
    r'on|off'
    return t

def t_TOKTARGET(t):
    r'target'
    return t

def t_TOKTEMPRATURE(t):
    r'temprature'
    return t

# 不做处理的符号 空格与tab
t_ignore = " \t"

# 行号统计
def t_newline(t):
    r'\n+'
    t.lexer.lineno += t.value.count("\n")

# 出错处理
def t_error(t):
    print "Illegal character '%s'" % t.value[0]
    t.lexer.skip(1)

# Build the parser
lexer = lex.lex()

# # 测试数据
# s = '''
# heat on
# '''

# # Give the lexer some input
# lexer.input(s)

# while True:
#     tok = lexer.token()
#     if not tok: break
#     print '(',tok.type,','+str(tok.value)+')'

```
接下来是在yacc文件中用产生式说明文法。
```python
#!/usr/bin/python
# -*- coding: utf-8 -*-  
# exam3.py

import ply.yacc as yacc
from exam3lex import tokens

def p_commands(p):
    '''commands : empty
                            | commands command
    '''

def p_command(p):
    '''command : heatswitch
                            | targetset'''

def p_heatswitch(p):
    'heatswitch : TOKHEAT STATE'
    print "Heat turned on or off"

def p_targetset(p):
    'targetset : TOKTARGET TOKTEMPRATURE NUMBER'
    print "temprature set"

def p_empty(p):
    'empty :'
    pass

# Error rule for syntax errors
def p_error(p):
    print "Syntax error in input!"

# Build the parser
parser = yacc.yacc()
 
while True:
   try:
       s = raw_input('calc > ')
   except EOFError:
       break
   if not s: continue
   result = parser.parse(s)
   print result
```

##拓展温度调节器使其可处理参数
利用参数p\[0\]获取属性值
```python
#!/usr/bin/python
# -*- coding: utf-8 -*-  
# exam4.py

import ply.lex as lex
import ply.yacc as yacc

tokens = ['NUMBER','TOKHEAT','STATE','TOKTARGET','TOKTEMPRATURE']

def t_NUMBER(t):
    r'[0-9]+'
    return t;

def t_TOKHEAT(t):
    r'heat'
    return t

def t_STATE(t):
    r'on|off'
    return t

def t_TOKTARGET(t):
    r'target'
    return t

def t_TOKTEMPRATURE(t):
    r'temprature'
    return t

# 不做处理的符号 空格与tab
t_ignore = " \t"

# 行号统计
def t_newline(t):
    r'\n+'
    t.lexer.lineno += t.value.count("\n")

# 出错处理
def t_error(t):
    print "Illegal character '%s'" % t.value[0]
    t.lexer.skip(1)

def p_commands(p):
    '''commands : empty
                            | commands command
    '''

def p_command(p):
    '''command : heat_switch
                            | target_set'''

def p_heatswitch(p):
    'heat_switch : TOKHEAT STATE'
    print "Heat turned " + p[2]

def p_targetset(p):
    'target_set : TOKTARGET TOKTEMPRATURE NUMBER'
    print "temprature set " + p[3]

def p_empty(p):
    'empty :'
    pass

# Error rule for syntax errors
def p_error(p):
    print "Syntax error in input!"


# Build the lexer
lexer = lex.lex()

# Build the parser
parser = yacc.yacc()
 
while True:
   try:
       s = raw_input('input > ')
   except EOFError:
       break
   if not s: continue
   result = parser.parse(s)
```

##解析配置文件
让我们继续讨论前面提到的配置文件：
```
zone "." {
        type hint;
        file "/etc/bind/db.root";
}
```
example 5:
```python
#!/usr/bin/python
# -*- coding: utf-8 -*-  
# exam5.py

import ply.lex as lex
import ply.yacc as yacc

#保留字
reserved = {
   'zone' : 'ZONETOK',
   'file' : 'FILETOK',
   'else' : 'ELSE',
}

tokens = ['WORD','FILENAME','QUOTE','OBRACE','EBRACE','SEMICOLON'] + list(reserved.values())

t_FILENAME = r'[a-zA-Z0-9/.-]+'

t_QUOTE = r'"'

t_OBRACE = r'{'

t_EBRACE = r'}'

t_SEMICOLON =  r';'

def t_WORD(t):
    r'[a-zA-Z][a-zA-Z0-9]+'
    t.type = reserved.get(t.value,'WORD')
    return t

# 不做处理的符号 空格与tab
t_ignore = " \t"

# 行号统计
def t_newline(t):
    r'\n+'
    t.lexer.lineno += t.value.count("\n")

# 出错处理
def t_error(t):
    print "Illegal character '%s'" % t.value[0]
    t.lexer.skip(1)


def p_commands(p):
    '''commands : empty 
                            | commands command
    '''
    if len(p) == 3:
        p[0] = p[2]

def p_command(p):
    'command : zone_set'
    p[0] = p[1]

def p_zoneset(p):
    'zone_set : ZONETOK quotename zonecontent'
    print "complete zone for",p[2],"found"
    p[0] = p[3]

def p_zonecontent(p):
    'zonecontent : OBRACE zonestatements EBRACE SEMICOLON'
    p[0] = p[2]

def p_quotename(p):
    'quotename : QUOTE FILENAME QUOTE'
    p[0] = p[2]

def p_zonestatements(p):
    '''zonestatements : empty
                                        | zonestatements zonestatement SEMICOLON
    '''
    if len(p) == 4:
        p[0] = p[2]


def p_zonestatement(p):
    '''zonestatement : statements
                                    | FILETOK quotename
    '''
    if p[1]=='file':
        p[0] = p[2]
        print "a zonefile name",p[2],"was encountered"

def p_block(p):
    'block : OBRACE zonestatements EBRACE SEMICOLON'

def p_statements(p):
    '''statements : empty
                            | statements statement 
    '''

def p_statement(p):
    '''statement : WORD
                            | block
                            | quotename'''

# Error rule for syntax errors
def p_error(p):
    print "Syntax error in input!"

def p_empty(p):
    'empty :'
    pass

# Build the lexer
lexer = lex.lex()

# Build the parser
parser = yacc.yacc()

file_object = open('test2.conf')
s = file_object.read()
print s

# lexer.input(s)

# while True:
#     tok = lexer.token()
#     if not tok: break
#     print tok

result = parser.parse(s)
print result
```



#深度阅读
GUN YACC (Bison)带有一个很常不错的info文件（.info），它是非常好的YACC语法文档，除了里面仅提到了一次Lex,其它的都还好。可以使用Emacs阅读info文件，或者非常不错的工具pinfo。


Flex有一个不错的用户手册，如果你已经理解Flex是做什么的，它还是非常有用的。


读完了这个Lex和YACC介绍，你可能想找到更多的信息。虽然以下的书我一本都没看过，不过听说不错：


1. Bision-The Yacc-Compatible Parser Generator

2. Lex&Yacc

3. Compliers: Principles,Techiniques,and Tools


Tohmas Niemann 写了一篇文档，讨论如何使用Lex和YACC写一个编译器和计算器。


usenet新闻组com.compilers也是非常有用的，不过请记住，那些人并非专门服务支持，在你发贴之前，阅读他们的感兴趣的页面，特别是FAQ


Lex-A Lexical Analyzer Generator\[[4](http://www.cs.utexas.edu/users/novak/lexpaper.htm)\]，M.E.Lesk and E.Schmidt，最原始的论文。

Yacc: Yet Another Compiler\[[5](http://www.cs.utexas.edu/users/novak/yaccpaper.htm)\]

#参考资料
以下推荐部分参考资料。
比以上内容略复杂的PLY示例项目\[[3](http://www.juanjoconti.com.ar/files/python/ply-examples/)\]中的calc.py感觉很适合作为进一步学习的资料。

\[1\] [Lex&YACC HOWTO 中文翻译](http://segmentfault.com/a/1190000000396608#articleHeader24)

\[2\] [PLY文档 中文翻译](http://www.pchou.info/open-source/2014/01/18/52da47204d4cb.html)

\[3\] [PLY示例项目](http://www.juanjoconti.com.ar/files/python/ply-examples/)

\[4\] [Lex-A Lexical Analyzer Generator](http://www.cs.utexas.edu/users/novak/lexpaper.htm)，M.E.Lesk and E.Schmidt

\[5\] [Yacc: Yet Another Compiler](http://www.cs.utexas.edu/users/novak/yaccpaper.htm)
