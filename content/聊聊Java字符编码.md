Date: 2013-09-29
Title: 聊聊Java字符串编码
Category: Java
Tags: String,Encode
Slug: java-string-encode                                

**声明：本文系博主原创，欢迎转载，但请注明作者和出处，谢谢！**

* [**引子**](#openning)

* [**Java中的字符编码**](#encoding)



## <a name="openning" id="openning"></a>引子

这几天尝试使用DES对产品中的某些字符串字段进行加密。本以为不会太费劲，但尝试的过程中却遇到的一些很有趣的问题，当然问题的原因就在本文的标题中：Java字符串编码。以前并没有对这方面内容有过关注，正好借这个机会总结一下。下面首先还是说说我遇到的问题。

我对一个字符串的byte数组进行DES加密后，再包装成字符串传到对端进行解密，解密当然也是针字符串的byte数组。解密过程出错，异常显示：`javax.crypto.BadPaddingException: Given final block not properly padded`。貌似是需要对最后一个block补齐，但真正的原因却出乎意料：在用String重新包装加密的byte数组的时候，String对其进行了编码，所以在解密端得到的byte数组已经不是加密后的byte数组了。下面先说说我的解决方案，对加密后的字符串，使用Base64转换成可见字符的字符串，而在对端解密前，先将可见字符串解密成原来的byte数组。由于Base64使用的可见字符在String编码的时候不会被改变，所以成功绕过了String的编码问题。

## <a name="encoding" id="encoding"></a>Java中的字符编码