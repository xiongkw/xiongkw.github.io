---
layout: post
title: Jenkins Jelly
categories: [编程, java]
tags: [jenkins]
---

> 一个`Jenkins`表单插件

#### 1. 菜单入口

`Jenkins`提供了`ManageLink、Action`等扩展点用于添加一个菜单项，这里直接继承`RootAction`

```java
@Extension
public class MyUserAction implements RootAction{

    public String getIconFileName() {
        return "setting.png";
    }

    public String getDisplayName() {
        return "My Users";
    }

    public String getUrlName() {
        return "myusers";
    }

}
```

> 运行插件可以看到左上角`Jenkins`菜单下多出了一个`My Users`菜单

#### 2. 准备数据源

这里使用`xml`文件作为数据源

```
public class MyUserAction implements RootAction{
    private List<MyUser> _all = new ArrayList<>();

    public List<Proxy> get_all() {
        return _all;
    }

    @Initializer(after = PLUGINS_PREPARED)
    public void load() throws IOException {
        String xml = getConfigFile().asString();
        _all = (List<MyUser>) Jenkins.XSTREAM.fromXML(xml);
    }

    private void store() {
        getConfigFile().write(_all);
    }

    private static XmlFile getConfigFile() throws IOException {
        File file = new File(Objects.requireNonNull(Jenkins.getInstanceOrNull()).getRootDir(), "myusers.xml");
        if (!file.exists()) {
            file.createNewFile();
        }
        return new XmlFile(file);
    }
}
```

```
public class MyUser{
    private String username;
    private String gender;
    private int age;
    private String phone;

    // g(s)etter...
}
```

#### 3. 编写列表页面
在`src/main/resources`目录下创建一个同名目录`MyUserAction`，并新建一个`index.jelly`文件

```
<?jelly escape-by-default='true'?>
<j:jelly xmlns:j="jelly:core" xmlns:st="jelly:stapler" xmlns:d="jelly:define" xmlns:l="/lib/layout" xmlns:t="/lib/hudson" xmlns:s="/lib/form">
<l:layout title="${%MyUsers}">
    <l:side-panel>
      <l:tasks>
        <l:task href="${rootURL}/" icon="icon-up icon-md" title="${%Back to Dashboard}"/>
        <l:task href="${rootURL}/manage" icon="icon-gear2 icon-md" permission="${app.ADMINISTER}" title="${%Manage Jenkins}"/>
        <l:task href="new" icon="icon-user icon-md" permission="${app.ADMINISTER}" title="${%New MyUser}"/>
      </l:tasks>
    </l:side-panel>
  <l:header>
    <style><!-- 这里可以定义css样式 -->
      
    </style>
  </l:header>
  <l:main-panel>
    <table id="users" class="sortable pane bigtable">
      <tr>
        <th initialSortDir="down" align="center">${%Username}</th>
        <th align="center">${%Age}</th>
        <th align="center">${%Gender}</th>
        <th align="center">${%Phone}</th>
        <th ></th>
      </tr>

      <j:forEach var="c" items="${it._all}"><!-- it代表MyUserAction实例，it._all表示调用get_all()方法 -->
        <tr id="user_${c.username}">
          <td align="center">${c.username}</td>
          <td align="center">${c.age}</td>
          <td align="center">${c.gender}</td>
          <td align="center">${c.phone}</td>

          <td align="center">
              <l:confirmationLink href="${rootURL}/myusers/deleteUser?username=${c.username}" message="确定要删除吗?" post="true">
                <l:icon class="icon-error icon-lg" tooltip="${%Delete}"/>
              </l:confirmationLink>
          </td>
        </tr>
      </j:forEach>

    </table>
  </l:main-panel>
</l:layout>
</j:jelly>

```

#### 4. 编写新增和删除接口

```
public class MyUserAction implements RootAction{

    @RequirePOST
    public synchronized void doCreateUser(StaplerRequest req, StaplerResponse rsp,
                                          @QueryParameter String username, @QueryParameter int age,
                                          @QueryParameter String gender, @QueryParameter String phone) throws IOException {
        MyUser user = new MyUser(username, age, gender, phone);
        _all.add(user);
        store();
    }

    public synchronized void doDeleteUser(StaplerRequest req, StaplerResponse rsp, @QueryParameter String username) throws IOException {
        _all.removeIf(u->username.equals(u.getUsername()));
        store();
    }

}
```

#### 5. 编写新增页面

new.jelly
```
<?jelly escape-by-default='true'?>
<j:jelly xmlns:j="jelly:core" xmlns:st="jelly:stapler" xmlns:d="jelly:define" xmlns:l="/lib/layout"
         xmlns:t="/lib/hudson" xmlns:f="/lib/form">
  <l:layout norefresh="true">
    <j:set var="descriptor" value="${it.descriptor}" />
    <l:main-panel>
      <f:form method="post" action="createUser" name="config">
        <f:entry title="${%Username}" field="username">
          <f:textbox name="username" clazz="required" checkMessage="用户名"/>
        </f:entry>
        <f:entry title="${%Age}">
          <f:textbox name="age" clazz="required" checkMessage="Age不可为空" default="${descriptor.defaultAge()}"/>
        </f:entry>
        <f:entry title="${%Gender}">
          <f:textbox name="gender" clazz="required" checkMessage="Gender不可为空"/>
            <select name="gender">
                <option value="0">女</option>
                <option value="1">男</option>
            </select>
        </f:entry>
        <f:entry title="${%Phone}" field="phone">
          <f:textbox name="phone" clazz="required" checkMessage="Phone不可为空"/>
        </f:entry>
        <f:block>
          <f:submit value="${%Save}"/>
        </f:block>
      </f:form>
    </l:main-panel>
  </l:layout>
</j:jelly>

```

#### 6. 编写Descriptor

编写`descriptor`实现后台校验和默认值处理

```
public class MyUserAction implements RootAction, Describable<MyUserAction> {

    public Descriptor<MyUserAction> getDescriptor() {
        return Objects.requireNonNull(Jenkins.getInstanceOrNull()).getDescriptorOrDie(MyUserAction.class);
    }

    @Extension
    public static class DescriptorImpl extends Descriptor<EadpAction> {

        public int defaultAge() {
            return 18;
        }

        public FormValidation doCheckUsername(@QueryParameter String value) {
            if (StringUtils.isBlank(value)) {
                return FormValidation.error("用户名不可为空");
            }
            for (MyUser u : _all) {
                if (value.equals(u.getUsername()) {
                    return FormValidation.error("用户 [" + value + "] 已存在");
                }
            }
            return FormValidation.ok();
        }

    }
}
```

#### 7. 参考

* [Extend Jenkins](https://wiki.jenkins.io/display/JENKINS/Extend+Jenkins)
* [Basic guide to Jelly usage in Jenkins](https://wiki.jenkins.io/display/JENKINS/Basic+guide+to+Jelly+usage+in+Jenkins)
* [Jelly taglib reference](https://reports.jenkins.io/core-taglib/jelly-taglib-ref.html)
* [Jelly form controls](https://wiki.jenkins.io/display/JENKINS/Jelly+form+controls)