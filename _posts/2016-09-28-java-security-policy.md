---
layout: post
title: Java安全策略与特权
categories: [编程, java]
tags: [security, policy, doPrivileged]
---

> 在 `Java` 中将执行程序分成本地和远程两种，本地代码默认视为可信任的，而远程代码则被看作是不受信的。对于授信的本地代码，可以访问一切本地资源。而对于非授信的远程代码，则无权访问本地资源

#### 1. Java的安全模型

> 虚拟机会把所有代码加载到不同的系统域和应用域，系统域部分专门负责与关键资源进行交互，而各个应用域部分则通过系统域的部分代理来对各种需要的资源进行访问。虚拟机中不同的受保护域 (`Protected Domain`)，对应不一样的权限 (`Permission`)。存在于不同域中的类文件就具有了当前域的全部权限

参考[Java 安全模型介绍](https://www.ibm.com/developerworks/cn/java/j-lo-javasecurity/)

`ProtectionDomain`：当类装载器将类型装入`Java`虚拟机时，它们将为每个类型指派一个保护域。保护域定义了授予一段特定代码的所有权限。（一个保护域对应策略文件中的一个或多个`Grant`子句。）装载入`Java`虚拟机的每一个类型都属于一个且仅属于一个保护域。

#### 2. 例子

一个读文件例子：假设`Main`为远程代码，`FileReader`为本地代码，以下通过不同的保护域(`ProtectionDomain`)来模拟

```
Main -> FileReader
```

`FileReader.java`
```java
public class FileReader {
    public static void read(String file) throws Exception {
        BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream(file)));
        System.out.println(br.readLine());
        br.close();
    }

    public static void privilegedRead(String file){
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    read(file);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return null;
            }
        });
    }
}
```

`Main.java`
```java
public class Main {
    public static void main(String[] args) throws Exception {
        System.out.println(Main.class.getProtectionDomain());
        System.out.println(FileReader.class.getProtectionDomain());
        String file = "D://resource/1.txt";
        FileReader.privilegedRead(file);
        FileReader.read(file);
    }
}
```

#### 3. 部署

把`Main`和`FileReader`放到不同的路径以模拟`本地和远程`

```
D:/test
        |my.policy
        |-lib/FileReader.class
        |-main/Main.class
```

安全策略`my.policy`，授权`FileReader`读权限，而`Main`则无权限

```
grant codeBase "file:/D:/test/lib/*" {
 permission java.io.FilePermission "D:/resource/*","read";
 permission java.lang.RuntimePermission "getProtectionDomain";
};

grant codeBase "file:/D:/test/main/*" {
 permission java.lang.RuntimePermission "getProtectionDomain";
};
```

#### 4. 运行

```
java -cp D:/test/main;D:/test/lib -Djava.security.manager -Djava.security.policy=D:/test/my.policy Main
```

> 可以看到这里用了`-Djava.security.manager`，也就是说平时我们用`java xxx.xxx`启动一个程序时是没有启用安全管理器的，通过测试也可以发现不启用时是可以运行通过的

运行结果，可以看到通过`doPrivileged`可以读取，但直接读会抛出`AccessControlException`异常

```
ProtectionDomain  (file:/D:/test/main/ <no signer certificates>)
 sun.misc.Launcher$AppClassLoader@1d16e93
 <no principals>
 java.security.Permissions@10dea4e (
 ("java.lang.RuntimePermission" "exitVM")
 ("java.io.FilePermission" "\D:\test\main\-" "read")
)


ProtectionDomain  (file:/D:/test/lib/ <no signer certificates>)
 sun.misc.Launcher$AppClassLoader@1d16e93
 <no principals>
 java.security.Permissions@1909752 (
 ("java.lang.RuntimePermission" "exitVM")
 ("java.io.FilePermission" "\D:\test\lib\-" "read")
)


bbb
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "D:\resource\1.txt"
 "read")
        at java.security.AccessControlContext.checkPermission(AccessControlContext.java:457)
        at java.security.AccessController.checkPermission(AccessController.java:884)
        at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
        at java.lang.SecurityManager.checkRead(SecurityManager.java:888)
        at java.io.FileInputStream.<init>(FileInputStream.java:127)
        at java.io.FileInputStream.<init>(FileInputStream.java:93)
        at FileReader.read(FileReader.java:14)
        at Main.main(Main.java:12)
```

#### 5. 源码分析

根据以上异常信息查看源码`FileInputStream.java:127`

```java
public FileInputStream(File file) throws FileNotFoundException {
    String name = (file != null ? file.getPath() : null);
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkRead(name);
    }
    open(name);
    // ...
}
```

> 可以看到`FileInputStream`中对指定文件做了读权限检查`security.checkRead(name)`

进一步到`SecurityManager`
```java
public void checkRead(String file) {
    checkPermission(new FilePermission(file,
        SecurityConstants.FILE_READ_ACTION));
}

public void checkPermission(Permission perm) {
    java.security.AccessController.checkPermission(perm);
}
```

`AccessController.checkPermission`
```java
public static void checkPermission(Permission perm) throws AccessControlException{
    if (perm == null) {
        throw new NullPointerException("permission can't be null");
    }
    AccessControlContext stack = getStackAccessControlContext();
    // if context is null, we had privileged system code on the stack.
    if (stack == null) {
        Debug debug = AccessControlContext.getDebug();
        boolean dumpDebug = false;
        if (debug != null) {
            dumpDebug = !Debug.isOn("codebase=");
            dumpDebug &= !Debug.isOn("permission=") ||
                Debug.isOn("permission=" + perm.getClass().getCanonicalName());
        }

        if (dumpDebug && Debug.isOn("stack")) {
            Thread.dumpStack();
        }

        if (dumpDebug && Debug.isOn("domain")) {
            debug.println("domain (context is null)");
        }

        if (dumpDebug) {
            debug.println("access allowed "+perm);
        }
        return;
    }

    AccessControlContext acc = stack.optimize();
    acc.checkPermission(perm);
}
```

`getStackAccessControlContext`是一个`native`方法，看注释返回的是方法调用栈的`ProtectionDomain`

> Returns the AccessControl context. i.e., it gets the protection domains of all the callers on the stack, starting at the first class with a non-null ProtectionDomain.   
> the access control context based on the current stack or null if there was only privileged system code.

`AccessControlContext.checkPermission`
```java
public void checkPermission(Permission perm)
    throws AccessControlException
{
    boolean dumpDebug = false;

    if (perm == null) {
        throw new NullPointerException("permission can't be null");
    }
    if (getDebug() != null) {
        // If "codebase" is not specified, we dump the info by default.
        dumpDebug = !Debug.isOn("codebase=");
        if (!dumpDebug) {
            // If "codebase" is specified, only dump if the specified code
            // value is in the stack.
            for (int i = 0; context != null && i < context.length; i++) {
                if (context[i].getCodeSource() != null &&
                    context[i].getCodeSource().getLocation() != null &&
                    Debug.isOn("codebase=" + context[i].getCodeSource().getLocation().toString())) {
                    dumpDebug = true;
                    break;
                }
            }
        }

        dumpDebug &= !Debug.isOn("permission=") ||
            Debug.isOn("permission=" + perm.getClass().getCanonicalName());

        if (dumpDebug && Debug.isOn("stack")) {
            Thread.dumpStack();
        }

        if (dumpDebug && Debug.isOn("domain")) {
            if (context == null) {
                debug.println("domain (context is null)");
            } else {
                for (int i=0; i< context.length; i++) {
                    debug.println("domain "+i+" "+context[i]);
                }
            }
        }
    }

    /*
     * iterate through the ProtectionDomains in the context.
     * Stop at the first one that doesn't allow the
     * requested permission (throwing an exception).
     *
     */

    /* if ctxt is null, all we had on the stack were system domains,
       or the first domain was a Privileged system domain. This
       is to make the common case for system code very fast */

    if (context == null) {
        checkPermission2(perm);
        return;
    }

    for (int i=0; i< context.length; i++) {
        if (context[i] != null &&  !context[i].implies(perm)) {
            if (dumpDebug) {
                debug.println("access denied " + perm);
            }

            if (Debug.isOn("failure") && debug != null) {
                // Want to make sure this is always displayed for failure,
                // but do not want to display again if already displayed
                // above.
                if (!dumpDebug) {
                    debug.println("access denied " + perm);
                }
                Thread.dumpStack();
                final ProtectionDomain pd = context[i];
                final Debug db = debug;
                AccessController.doPrivileged (new PrivilegedAction<Void>() {
                    public Void run() {
                        db.println("domain that failed "+pd);
                        return null;
                    }
                });
            }
            throw new AccessControlException("access denied "+perm, perm);
        }
    }

    // allow if all of them allowed access
    if (dumpDebug) {
        debug.println("access allowed "+perm);
    }

    checkPermission2(perm);
}
```

> 按顺序检查`AccessControlContext`中的每一个`ProtectionDomain`

`ProtectionDomain.implies`
```java
public boolean implies(Permission permission) {

    if (hasAllPerm) {
        // internal permission collection already has AllPermission -
        // no need to go to policy
        return true;
    }

    if (!staticPermissions &&
        Policy.getPolicyNoCheck().implies(this, permission))
        return true;
    if (permissions != null)
        return permissions.implies(permission);

    return false;
}
```

那为什么`AccessController.doPrivileged`能绕过安全检查呢？

`AccessController.doPrivileged`是一个`native`方法：
```java
public static native <T> T doPrivileged(PrivilegedAction<T> action);
```

看其注释
```
Performs the specified PrivilegedAction with privileges enabled. The action is performed with all of the permissions possessed by the caller's protection domain.
If the action's run method throws an (unchecked) exception, it will propagate through this method.
Note that any DomainCombiner associated with the current AccessControlContext will be ignored while the action is performed.
```

意思是使用调用方的权限来执行，我们的例子里，使用的就是`FileReader`的权限