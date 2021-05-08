# 最全 Yaml 语法详解


## 简单说明

Yaml 是一个可读性高，用来表达数据序列的格式。Yaml 的意思是：仍是一种标记语言，强调这种语言以数据做为中心，而不是以标记语言为重点

## 基本语法

- 缩进时不允许使用 Tab 键，只允许使用空格
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
- 标识注释，从这个字符一直到行尾，都会被解释器忽略

## Yaml 支持的数据结构

- 对象：键值对的集合，又称为映射mapping<ruby>映射<rt>mapping</rt></ruby>/ 哈希hash<ruby>哈希<rt>hash</rt></ruby>/ 字典dictionary<ruby>字典<rt>dictionary</rt></ruby>
- 数组：一组按次序排列的值，又称为序列sequence<ruby>序列<rt>sequence</rt></ruby>/ 列表list<ruby>列表<rt>list</rt></ruby>
- 纯量scalars<ruby>纯量<rt>scalars</rt></ruby>：单个的、不可再分的值

### 对象类型：对象的一组键值对，使用冒号结构表示

```yaml
name: Oliver
age:24
```

Yaml 也允许另一种写法，将所有键值对写成一个行内对象

```yaml
hash: {name: Oliver, age: 24}
```

Yaml 允许存在层级键值对

```yaml
major:
 name: computer
 name: law
```

### 数组类型：一组连词线开头的行，构成一个数组

```yaml
Sport:
 - Swim
 - Run
```

数组也可以采用行内表示法

```yaml
Sport: [Swim, Run]
```

数组的一个相对复杂的实现方式

```yaml
products:
 -
  id: 1
  name: egg
  price: 8
 -
  id: 2
  name: meat
  price: 33
```

### 复合结构：对象和数组可以结合使用，形成复合结构

```yaml
sport:
 - swim
 - run
 - basketball

city:
 haidian: beijing.haidian
 jinan: shandong.jinan
 hefei: anhui.hefei
```

### 纯量：纯量是最基本的、不可再分的值

1. 字符串、布尔值、整数、浮点数、Null
2. 时间、日期

**数值**

数值直接以字面量的形式表示

```yaml
number: 12.30
```

**布尔值**

布尔值用 true 和 false 表示

```yaml
isSet: true
```

**Null**

null 用～表示

```yaml
parent: ~
```

**时间**

时间采用 ISO8601 格式

```yaml
iso8601: 2001-12-14t21:59:43.10-05:00
```

**日期**

日期采用 ISO8601 格式的年、月、日表示

```yaml
date: 1976-07-31
```

**类型转换**

Yaml 允许使用两个感叹号，强制转换数据类型

```yaml
e: !!str 123
f: !!str true
```

**字符串**

字符串默认不使用引号表示

```yaml
str: 我是字符串
```

如果字符串之中包含空格或特殊字符，需要放在引号中

```yaml
str: '内容： 字符串'
```

单引号之中如果还有单引号，必须连续使用两个单引号转义

```yaml
str: 'labor’‘s day'
```

字符串可以写成多行，从第二行开始，必须有一个单空格缩进，换行符会被转为空格

```yaml
str: 我是一段
 多行
 字符串
```

多行字符串可以使用 | 保留换行符，也可以使用 > 折叠换行

```yaml
this: |
 Foo
 Bar
that: >
 Foo
 Bar
```

\+ 表示保留文字块末尾的换行，- 表示删除字符串末尾的换行

```yaml
s1: |
 Foo

s2: |+
 Foo

s3: |-
 Foo
```

## 引用

& 锚点和 * 别名，可以用来引用:

```yaml
defaults: &defaults
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  <<: *defaults

test:
  database: myapp_test
  <<: *defaults
```

相当于

```yaml
defaults:
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  adapter:  postgres
  host:     localhost

test:
  database: myapp_test
  adapter:  postgres
  host:     localhost
```

& 用来建立锚点（defaults），<< 表示合并到当前数据，* 用来引用锚点。
