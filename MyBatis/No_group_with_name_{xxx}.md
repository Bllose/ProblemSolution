# 现象与前提
```
### Error updating database.  Cause: java.lang.IllegalArgumentException: No group with name {deviceSn}
### The error may exist in com/huawei/it/cbg/ccp/tbsd/asc/dao/AscRequestLogDAO.xml
### The error may involve com.huawei.it.cbg.ccp.tbsd.asc.dao.AscRequestLogDAO.insertSelective-Inline
### The error occurred while setting parameters
### SQL: insert into asc_request_log_t      ( url,                       request_type,                       request_param,                       response_data,                                     process_flag )       values ( ?,                       ?,                       ?,                       ?,                                     ? )
### Cause: java.lang.IllegalArgumentException: No group with name {deviceSn}
	at org.apache.ibatis.exceptions.ExceptionFactory.wrapException(ExceptionFactory.java:30) ~[mybatis-3.5.6.jar:3.5.6]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.update(DefaultSqlSession.java:199) ~[mybatis-3.5.6.jar:3.5.6]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.insert(DefaultSqlSession.java:184) ~[mybatis-3.5.6.jar:3.5.6]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_262]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_262]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_262]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_262]
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:426) ~[mybatis-spring-2.0.2.jar:2.0.2]
	... 40 more
Caused by: java.lang.IllegalArgumentException: No group with name {deviceSn}
	at java.util.regex.Matcher.appendReplacement(Matcher.java:849) ~[?:1.8.0_262]
	at java.util.regex.Matcher.replaceFirst(Matcher.java:1004) ~[?:1.8.0_262]
	at java.lang.String.replaceFirst(String.java:2178) ~[?:1.8.0_262]
	at com.huawei.it.cbg.ccp.tbsd.asc.interceptor.MybatisShowSqlInterceptor.getSql(MybatisShowSqlInterceptor.java:133) ~[classes/:?]
	at com.huawei.it.cbg.ccp.tbsd.asc.interceptor.MybatisShowSqlInterceptor.intercept(MybatisShowSqlInterceptor.java:89) ~[classes/:?]
	at org.apache.ibatis.plugin.Plugin.invoke(Plugin.java:61) ~[mybatis-3.5.6.jar:3.5.6]
	at com.sun.proxy.$Proxy318.update(Unknown Source) ~[?:?]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.update(DefaultSqlSession.java:197) ~[mybatis-3.5.6.jar:3.5.6]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.insert(DefaultSqlSession.java:184) ~[mybatis-3.5.6.jar:3.5.6]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_262]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_262]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_262]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_262]
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:426) ~[mybatis-spring-2.0.2.jar:2.0.2]
	... 40 more
```
排查自己的SQL语句，入参对象， 均没有报错信息中的```deviceSn```。

# 根本原因
MyBatis 在尝试将参数注入到 SQL 语句的占位符 ```?``` 时， 会将参数先处理一次，在处理的过程中， 如果这个入参包含 ${any_name} 字符， 便会尝试查找父类的 ```Pattern```中是否包含此占位符。  
但在当前的逻辑场景中，应该是将现成的 参数 注入 占位符 中， 所以父类的```Pattern```肯定只有占位符```\?```, 导致报错.

本次异常时，导致报错的**直接原因**（带有${}符号的请求报文）:  
```
{
	  ...
    "returnApplyDetailVo": [
        {
            "deviceSn": "${deviceSn}",
            "prodTypeCode": "OFFE00023017",
            "prodMcMode": "51090XKN",
            ...
          }
     ]
}
```

根本原因：
java.util.regex.Matcher.appendReplacement(StringBuffer sb, String replacement)
代码片段如下：
``` JAVA
        while (cursor < replacement.length()) {
            char nextChar = replacement.charAt(cursor);
            if (nextChar == '\\') {
                cursor++;
                if (cursor == replacement.length())
                    throw new IllegalArgumentException(
                        "character to be escaped is missing");
                nextChar = replacement.charAt(cursor);
                result.append(nextChar);
                cursor++;
            } else if (nextChar == '$') {
                // Skip past $
                cursor++;
                // Throw IAE if this "$" is the last character in replacement
                if (cursor == replacement.length())
                   throw new IllegalArgumentException(
                        "Illegal group reference: group index is missing");
                nextChar = replacement.charAt(cursor);
                int refNum = -1;
                if (nextChar == '{') {
                    cursor++;
                    StringBuilder gsb = new StringBuilder();
                    while (cursor < replacement.length()) {
                        nextChar = replacement.charAt(cursor);
                        if (ASCII.isLower(nextChar) ||
                            ASCII.isUpper(nextChar) ||
                            ASCII.isDigit(nextChar)) {
                            gsb.append(nextChar);
                            cursor++;
                        } else {
                            break;
                        }
                    }
                    if (gsb.length() == 0)
                        throw new IllegalArgumentException(
                            "named capturing group has 0 length name");
                    if (nextChar != '}')
                        throw new IllegalArgumentException(
                            "named capturing group is missing trailing '}'");
                    String gname = gsb.toString();
                    if (ASCII.isDigit(gname.charAt(0)))
                        throw new IllegalArgumentException(
                            "capturing group name {" + gname +
                            "} starts with digit character");
                    if (!parentPattern.namedGroups().containsKey(gname))
                    /**
                    * 抛出异常的具体位置
                    */
                        throw new IllegalArgumentException(
                            "No group with name {" + gname + "}");
                    refNum = parentPattern.namedGroups().get(gname);
                    cursor++;
                } else {
                    // The first number is always a group
                    refNum = (int)nextChar - '0';
                    if ((refNum < 0)||(refNum > 9))
                        throw new IllegalArgumentException(
                            "Illegal group reference");
                    cursor++;
                    // Capture the largest legal group string
                    boolean done = false;
                    while (!done) {
                        if (cursor >= replacement.length()) {
                            break;
                        }
                        int nextDigit = replacement.charAt(cursor) - '0';
                        if ((nextDigit < 0)||(nextDigit > 9)) { // not a number
                            break;
                        }
                        int newRefNum = (refNum * 10) + nextDigit;
                        if (groupCount() < newRefNum) {
                            done = true;
                        } else {
                            refNum = newRefNum;
                            cursor++;
                        }
                    }
                }
                // Append group
                if (start(refNum) != -1 && end(refNum) != -1)
                    result.append(text, start(refNum), end(refNum));
            } else {
                result.append(nextChar);
                cursor++;
            }
        }
```     
