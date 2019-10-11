---
title: 'Angular: The First Glance'
date: 2019-07-07 19:23:26
tags: ['Angular']
---

在前端三大框架Angular、React、Vue之中，Angular始终是最特殊的一个。或者说，在我看来，Angular很完整的体现了Google对于技术革新的激进追求。

Angular在很早就激进的转向来Ts，而从目前来看，这是一个超前时代一步的选择，从目前来看，React与Vue都不约而同的转向了Ts，追求约束更强、更工程化的前端体系。而相对于React和Vue来说，Ng是更加符合框架这个名字的。从构建、路由、状态管理到测试，Ng自带的组件基本都可以满足需求，而不需要另外的第三方库。

在这篇博客中，主要来粗浅的领略一下Ng的强大之处，简单的梳理一下Ng的脉络。

# 起点：app.module.ts

一个典型的app.module.ts文件如下所示：

```
@NgModule({
  declarations: [
    AppComponent,
    HeroesComponent,
    HeroDetailComponent,
    MessagesComponent,
    DashboardComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    AppRoutingModule,
    HttpClientModule,
    HttpClientInMemoryWebApiModule.forRoot(
      InMemoryDataService, { dataEncapsulation: false }
    )
  ],
  
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

我们可以看到，实际上就是一个使用了@NgModule Decorator的Object。而我们需要关注的就是其中的几个key-value：

* declarations
在这里声明该模块（module）所使用的组件（component）

* imports
在这里声明该模块中需要用到的其他模块。从Ng架构上来看，一般是app module下有很多子module，而每个子module可能含有其他子module，或者由很多component所组成。类似于以下架构：

```
--app(module)
----home(module)
------homeHeader(component)
------homeContent(component)
----auth(module)
------authModal(component)
```

* providers
声明在该module中定义的、可于其他地方使用的services

* bootstrap
用于在root module中声明main application view

* exports
可以被其他module引用的，在declarations中声明的component

可以使用angular-cli很方便的新建一个module：

```
ng generate module moduleName
```

# 基础：Component组件
在Ng中，最基本的组成单元就是component，由component——>module——>app构成了一个Ng的SPA。而在Ng中可以很方便的通过angular-cli新建一个component：

```
ng generate component componentName
```

将会生成一个与comonentName同名的文件夹，这里假设使用的name为dashboard，生成的内容如下：
```
--dashboard(文件夹)
----dashboard.component.css
----dashboard.component.html
----dashboard.component.spec.ts
----dashboard.component.ts
```

* dashboard.component.css
在这里可以定义一个component的样式表，与其他框架下相同。

* dashboard.component.html
在这里定义component的html组成，通过ng directive在此设定相应的事件，以及通过模板的方式引用变量值

* dashboard.component.spec.ts
Ng component的单元测试文件

* dashboard.component.ts
在此声明变量，进行依赖注入，写相应的事件响应函数等。

可以说，组成一个dashboard component最重要的就是dashboard.component.html和dashboard.component.ts这两个文件。下面首先看看其中的.ts文件：

```
import { Component, OnInit } from '@angular/core';
import { Hero } from '../hero';
import { HeroService } from '../hero.service';

@Component({
  selector: 'app-dashboard',
  templateUrl: './dashboard.component.html',
  styleUrls: [ './dashboard.component.css' ]
})
export class DashboardComponent implements OnInit {
  heroes: Hero[] = [];

  constructor(private heroService: HeroService) { }

  ngOnInit() {
    this.getHeroes();
  }

  getHeroes(): void {
    this.heroService.getHeroes()
      .subscribe(heroes => this.heroes = heroes.slice(1, 5));
  }
}
```

从直观上来看，component组件就是使用了@Component decorator的class object。首先看传至decorator的object的三个参数：

* selector 这里指明了在html模板中使用该组件时使用的名字，如<app-dashboard></app-dashboard>
* templateUrl 该组件对应的模板的相对路径
* styleUrls 该组件对应的样式表的相对路径

随后注意到，这里的DashboardComponent implement了OnInit这个Interface。这里的OnInit是Ng的组件生命周期回调函数，也就是在组件初始化时执行。要实现其他生命周期函数，只需要import进来并implement即可。Ng的生命周期函数很多，需要另外专门写成一篇来讲，在此就不展开了。

在constructor中进行依赖注入，如这里将heroService注入，即可在其他地方通过this.heroService进行调用了。然后看.html中模板的写法：

```
<h3>Top Heroes</h3>
<div class="grid grid-pad">
  <a *ngFor="let hero of heroes" class="col-1-4"
    routerLink="/detail/{{hero.id}}">
    <div class="module hero">
      <h4>{{hero.name}}</h4>
    </div>
  </a>
</div>
```

在这里可以像以往一样定义html tag的class和id。同时，这里使用了*ngFor这样的directives（指令），将.ts文件中声明的heros在模板中使用并map成节点。Ng提供了非常多的指令用来做事件响应函数定义、在模板中使用变量等，需要另外专门写上一篇博客来讲。

至此，我们已经可以利用Ng来打造一些简单的东西了，而更复杂的话题包括生命周期、指令、事件响应、路由、测试等，就放在以后再慢慢深入了。