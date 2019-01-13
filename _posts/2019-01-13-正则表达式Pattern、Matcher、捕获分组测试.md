---
layout:     post
title:      正则表达式Pattern、Matcher、捕获分组测试
subtitle:   Pattern、Matcher
date:       2019-01-13
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
---

# 正则表达式测试

`Pattern`、`Matcher`等与正则表达式相关类的文档见[JDK API文档](https://docs.oracle.com/javase/8/docs/api/java/util/regex/Pattern.html)，以及下面注释的博客链接。

###  一、`Pattern`类测试

```java
import java.util.Arrays;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Pattern/Matcher 测试
 * 参考博客：
 * https://my.oschina.net/CasparLi/blog/361859
 * https://winter8.iteye.com/blog/1463244
 */
public class PatternTest {
    public static void main(String[] args) {
        // \w+ 匹配一个或多个字母、数字、下划线
        Pattern p1 = Pattern.compile("\\w+");
        // 正则表达式
        String patternString = p1.pattern();
        System.out.println(patternString);

        // \d+ 匹配一个或多个数字 Pattern.split分割
        Pattern p2 = Pattern.compile("\\d+");
        String[] str = p2.split("我的QQ是:456456我的电话是:0532214我的邮箱是:aaa@aaa.com");
        System.out.println(Arrays.toString(str));

        // Pattern.matches 匹配全部，才返回true
        System.out.println(Pattern.matches("\\d+", "23333"));
        System.out.println(Pattern.matches("\\d+", "23333aa"));
        System.out.println(Pattern.matches("\\d+", "233bb33"));

        Matcher matcher = p2.matcher("222333bb44");
        // 返回p 也就是返回该Matcher对象是由哪个Pattern对象的创建的
        System.out.println(matcher.pattern());

        // Matcher.matches()  匹配全部，才返回true
        Matcher m1 = p2.matcher("22bb33");
        // 返回false,因为bb不能被\d+匹配,导致整个字符串匹配未成功.
        System.out.println("Matcher.matches(): " + m1.matches()/* + ", start: " + m1.start() + ";end: " + m1.end()*/);
        // 返回true,因为\d+匹配到了整个字符串
        Matcher m2 = p2.matcher("2233");
        System.out.println("Matcher.matches(): " + m2.matches() + ", start: " + m2.start() + ";end: " + m2.end());

        // Matcher.lookingAt() 字符串开始处开始匹配
        Matcher m3 = p2.matcher("22bb333");
        // 返回true,因为\d+匹配到了前面的22
        System.out.println("Matcher.lookingAt(): " + m3.lookingAt() + ", start: " + m3.start() + ";end: " + m3.end());
        // 返回false,因为\d+不能匹配前面的bb
        Matcher m4 = p2.matcher("bb2233");
        System.out.println("Matcher.lookingAt(): " + m4.lookingAt()/* + ", start: " + m4.start() + ";end: " + m4.end()*/);

        // Matcher.find() 匹配到的字符串可以在任何位置
        Matcher m5 = p2.matcher("22bb23");
        System.out.println("m5.find(): " + m5.find());
        Matcher m6 = p2.matcher("aa2223");
        System.out.println("m6.find(): " + m6.find());
        Matcher m7 = p2.matcher("aa2223bb");
        System.out.println("m7.find(): " + m7.find());
        Matcher m8 = p2.matcher("aabb");
        System.out.println("m8.find(): " + m8.find());
    }
}
/*输出：
\w+
[我的QQ是:, 我的电话是:, 我的邮箱是:aaa@aaa.com]
true
false
false
\d+
Matcher.matches(): false
Matcher.matches(): true, start: 0;end: 4
Matcher.lookingAt(): true, start: 0;end: 2
Matcher.lookingAt(): false
m5.find(): true
m6.find(): true
m7.find(): true
m8.find(): false
*/
```

### 二、`Matcher`类测试

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Matcher 测试
 */
public class MatcherTest {
    private static final Pattern PATTERN = Pattern.compile("\\d+");

    public static void main(String[] args){
        // find() 对字符串进行匹配,匹配到的字符串可以在任何位置
        Matcher m = PATTERN.matcher("aaa2223bb");
        System.out.println("m.find(): " + m.find());
        System.out.println("m.start(): " + m.start());
        System.out.println("m.end(): " + m.end());
        System.out.println("m.group(): " + m.group());

        System.out.println("--------------------");

        // lookingAt() 对前面的字符串进行匹配,只有匹配到的字符串在最前面才返回true 
        Matcher m2 = PATTERN.matcher("2223bb");
        System.out.println("m2.lookingAt(): " + m2.lookingAt());
        System.out.println("m2.start(): " + m2.start());
        System.out.println("m2.end(): " + m2.end());
        System.out.println("m2.group(): " + m2.group());

        System.out.println("---------------------");

        // matches()
        Matcher m3 = PATTERN.matcher("2223bb");
        // matches() 对整个字符串进行匹配,只有整个字符串都匹配了才返回true
        System.out.println("m3.matches(): " + m3.matches());
        // System.out.println("m3.start(): " + m3.start()); // 异常
        // System.out.println("m3.end(): " + m3.end()); // 异常
        // System.out.println("m3.group(): " + m3.group()); // 异常
    }
}
/*输出:
m.find(): true
m.start(): 3
m.end(): 7
m.group(): 2223
--------------------
m2.lookingAt(): true
m2.start(): 0
m2.end(): 4
m2.group(): 2223
---------------------
m3.matches(): false
*/
```

###  三、正则表达式(捕获分组)测试

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * 正则表达式分组group测试
 */
public class GroupTest {

    private static final Pattern PATTERN_ONE = Pattern.compile("([a-z]+)(\\d+)");
    private static final Pattern PATTERN_TWO = Pattern.compile("(\\d+)");

    public static void main(String[] args) {
        Matcher m = PATTERN_ONE.matcher("aaa2223bb111");

        // 对字符串进行匹配,匹配到的子字符串序列可以在任何位置
        while (m.find()) {
            System.out.println("***********");
            // group() 等于 group(0) 表示匹配的子字符串序列
            System.out.println("m.group(0): " + m.group(0));
            System.out.println("m.start(0): " + m.start(0) + ", m.end(1): " + m.end(0));
            // gropu(1) 正则表达式第一个括号，start(1)、end(1)与group(1)对应
            System.out.println("m.group(1): " + m.group(1));
            System.out.println("m.start(1): " + m.start(1) + ", m.end(1): " + m.end(1));
            // gropu(2) 正则表达式第二个括号
            System.out.println("m.group(2): " + m.group(2));
            System.out.println("m.start(2): " + m.start(2) + ", m.end(2): " + m.end(2));
        }

        // 2
        System.out.println("m.groupCount(): " + m.groupCount());
        // false
        System.out.println("m.find(): " + m.find());
        // System.out.println("m.group(0): " + m.group(0));
        // System.out.println("m.start(0): " + m.start(0) + ", m.end(1): " + m.end(0));
        // System.out.println("m.group(1): " + m.group(1));
        // System.out.println("m.start(1): " + m.start(1) + ", m.end(1): " + m.end(1));
        // System.out.println("m.group(2): " + m.group(2));
        // System.out.println("m.start(2): " + m.start(2) + ", m.end(2): " + m.end(2));

        System.out.println("--------------------------");
        Matcher m2 = PATTERN_TWO.matcher("我的QQ是:456456 我的电话是:0532214 我的邮箱是:aaa123@aaa.com");
        while (m2.find()) {
            System.out.println(m2.group() + ", start: " + m2.start() + ",end: " + m2.end());
        }
    }
}
/*输出:
***********
m.group(0): aaa2223
m.start(0): 0, m.end(1): 7
m.group(1): aaa
m.start(1): 0, m.end(1): 3
m.group(2): 2223
m.start(2): 3, m.end(2): 7
***********
m.group(0): bb111
m.start(0): 7, m.end(1): 12
m.group(1): bb
m.start(1): 7, m.end(1): 9
m.group(2): 111
m.start(2): 9, m.end(2): 12
m.groupCount(): 2
m.find(): false
--------------------------
456456, start: 6,end: 12
0532214, start: 19,end: 26
123, start: 36,end: 39
*/
```

