---
title: Vue学习笔记(二)
date: 2023-09-06 16:17:03
categories: 
- 学习笔记
tags:
- 前端
- Vue
cover: ../images/Vue学习笔记/vuelogo.png
---

[vite-vue-admin](https://github.com/peng-xiao-shuai/vite-vue-admin)项目学习



## 左侧菜单栏

左边菜单栏的定义由两部分控制，一个是接口 admin/info返回的 data.data.menus。另一个是路由router/index.ts的addRouter中。

admin/info：data.data.menus这个接口用于以用户为粒度，控制左侧菜单栏。在每次登陆后返回该用户可访问的菜单栏。

addRouter中则是菜单栏中各种子项的路由定义。

也就是说如果想要在左侧显示自定义的菜单栏，二者缺一不可。



在加载的时候stores/user.ts 路径下的menusFilter函数会结合两者(靠name来匹配，所以name唯一定义)，在这里构建最终的菜单栏。其中id，title，icon以data.data.menus返回为准，方便自定义菜单样式。

```typescript
      item.meta.id = each.id
      if (each.title) {
        item.meta.title = each.title
      }
      if (each.icon) {
        item.meta.icon = each.icon
```





https://www.skypack.dev/view/el-plus-powerful-table-ts

