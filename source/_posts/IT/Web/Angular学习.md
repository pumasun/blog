---
title: Angular学习
date: 2023-09-04 11:28:37
tags: ['Web', 'Angular', 'TypeScript']
description: 'Angular学习笔记'
keywords: ['Web', 'Guide', 'Angular']
category: ['Web']
---
[官网En](https://angular.io/)
[官网Zh](https://angular.cn/)

# 版本
最早接触Angular的版本是1.x，各种无法解决的问题与繁琐的写法。
再一次进入我视野的时候，Angular的版本已经更新到 v16。
既然仍然可以与React和Vue抗衡，应该值的一看。

# 开始
## 搭建环境
不再赘述，参考[链接](https://angular.cn/guide/setup-local)

## 看例子 + 视频
我看了，可以快速建立大体印象
[教程](https://angular.cn/tutorial/first-app)
[视频_Youtube](https://www.youtube.com/playlist?list=PL1w1q3fL4pmj9k1FrJ3Pe91EPub2_h4jF)



# 组件
## 组件生命周期
1. `ngOnChanges`	当输入或输出绑定值更改时。
2. `ngOnInit`	在第一个 ngOnChanges 之后。
3. `ngDoCheck`	开发人员的自定义变更检测。
4. `ngAfterContentInit`	组件内容初始化后。
5. `ngAfterContentChecked`	在每次检查组件内容之后。
6. `ngAfterViewInit`	在组件的视图被初始化之后。
7. `ngAfterViewChecked`	在每次检查组件视图之后。
8. `ngOnDestroy` 在指令被销毁之前。

## 定义
``` TypeScript
@Component({
    selector: 'my-component',
    template: ``,
    style: []
})
export class MyComponent{
    constructor(){}
}
```
## 模板
### 如何定义
1. 通过`template`指定模板字符串。适合内容简单的组件。
2. 通过`templateUrl`指定HTML文件，提供模板信息。适合内容比较复杂的组件。

### 模板语法

#### 模板表达式
类似于 JavaScript 表达式

不支持

- 赋值（`=`, `+=`, `-=`, `...`）
- 运算符，比如 `new`、`typeof` 或 `instanceof` 等。
- 串联表达式`;`或`,`
- 自增和自减运算符：`++` 和 `--`
- 一些 ES2015+ 版本的运算符
- 位运算，比如 `|` 和 `&`
- 

额外支持基本赋值(`=`)和分号(`;`)串联表达式
- 例如在`*ngTemplateOutlet`表达式中，可以使用`;`串联的表达式
  ``` HTML
  <ng-container *ngTemplateOutlet="sk; context: myContext"></ng-container>
  ```



#### 文本插值
使用双花括号`{{`和`}}`包裹的表达式嵌入到**文本**中。
其中，表达式可以为组件的属性，也可以为函数调用。
> TODO 是否支持其他的表达式？有什么限制？

#### 模板上下文
模板只能引用模板上下文中的内容，通常是组件实例。

- **输入变量**
  ``` HTML
  <tag [attr]="componentField1" (event)="componentMethod()">{{componentField2}}</tag>
  ```
- **语法变量** 
  - 组件的 事件处理 方法中，可以使用模板上下文变量 `$event` 用作参数
    ``` HTML
    <button type="button" (click)="onSave($event)">Save</button>
    ```
  - `ngFor`的表达式中，会创建遍历用临时变量
    ``` HTML
    <button type="button" *ngFor="let item of items" (click)="clickAction(item)">
    ```
- **引用变量** 使用`#name`标记的标签，其组件实例会被作为上下文变量
  ``` HTML
  <form #formName (ngSubmit)="onSubmit(formName)"> ... </form>
  ```

#### Attribute绑定
用于设置无障碍性访问、动态设置样式属性等场景。

Attribute绑定以`attr`作为前缀，`.`作为连接，后面指定要设定的attribute名称。
``` HTML
<button type="button" [attr.aria-label]="actionName">{{actionName}} with Aria</button>
<td [attr.colspan]="spanNum">...<td>
```

#### Class绑定
用于动态设定CSS类

- 单一类绑定, 通过boolean表达式决定是否添加某个样式类
  ``` HTML
  <div [class.styleClassName]="booleanExpOrField">...</div>
  ```
- 多重类绑定
  - `string`表达式，由空格分隔的多个样式类组成的字符串
  - `string[]`表达式，由样式类字符串组成的字符串数组
  - `Record<string,boolean>`表达式，由样式类作为key,boolean作为值的对象，当值为true是适用样式类

#### Style绑定
用于动态设定CSS属性

- 单一样式绑定
  ``` HTML
  <div [style.width]="100px"></div>
  ```
- 带单位的单一样式绑定
  ``` HTML
  <div [style.width.px]="100"></div>
  ```
- 多重样式绑定，分号`;`分隔的字符串
  ```HTML
  <div [style]="width: 100px; height: 100px"></div>
  ```
- 多重样式绑定，`Record<string,boolean>`表达式
  ``` HTML
  <div [style]="{width: '100px', height: '100px'}"></div>
  ```

#### 事件绑定
事件绑定使用括号(`(`和`)`)标识监听的事件名称，属性值为事件响应表达式，并使用`$event`变量名表示事件参数
``` HTML
<button (click)="clickAction($event)">...</button>
```

##### 绑定到键盘事件

适用`keydown`关键字，可以绑定键盘事件。
``` HTML
<input (keydown.shift.t)="onKeydown($event)" />
```
键盘事件可以分为两种,如果不指定类型，则默认绑定到键值:
- 键值绑定
  
  `keydown.key.`+以点号`.`分隔的键值组合
- 键码绑定
  
  在不同操作系统或者区分修饰键左右的情况下，键值绑定可能无法满足需要，此是可以使用键码绑定。

  `keydown.code.`+以点号`.`分隔的键码组合


#### 双向绑定
Angular 的双向绑定语法是方括号和圆括号的组合 `[()]`。`[]` 进行属性绑定，`()` 进行事件绑定
``` HTML
<app-sizer [(size)]="fontSizePx"></app-sizer>
```
等价于
```HTML
<app-sizer [size]="fontSizePx" (sizeChange)="fontSizePx=$event"></app-sizer>
```
为了使双向数据绑定有效，`@Output()` 属性的名字必须遵循 `inputChange` 模式，其中 `input` 是相应 `@Input()` 属性的名字。比如，如果 `@Input()` 属性为 `size`，则 `@Output()` 属性必须为 `sizeChange`。
``` TypeScript
@Input()  size!: number | string;
@Output() sizeChange = new EventEmitter<number>();
```
#### 管道
管道是在模板表达式中使用的简单函数，用于接受输入值并返回转换后的值。
可以使用管道来转换字符串、货币金额、日期和其他数据以进行显示。

##### 管道参数

有些管道可以接收参数，比如日期管道(`date`)可以接收日期格式
``` HTML
<p>My Birthday is {{ birthday | date:"MM/dd/yy" }} </p>
```

##### 管道串联

可以对管道进行串联，以便一个管道的输出成为下一个管道的输入。
```HTML
<p>My Birthday is {{ birthday | date:"MM/dd/yy" | uppercase }} </p>
```

##### 常用内置管道
- `AsyncPipe`	
  从一个异步回执中解出一个值。
- `CurrencyPipe`	
  将数字转换为货币字符串，根据确定组大小和分隔符、小数点字符和其他特定于区域设置的配置的区域设置规则进行格式化。
- `DatePipe`	
  根据区域设置规则格式化日期值。
- `DecimalPipe`	
  根据数字选项和区域设置规则格式化值。区域设置确定组的大小和分隔符、小数点字符和其他特定于区域设置的配置。
- `I18nPluralPipe`	
  将值映射到根据语言环境规则对该值进行复数化的字符串。
- `I18nSelectPipe`	
  通用选择器，用于显示与当前值匹配的字符串。
- `JsonPipe`	
  把一个值转换成 JSON 字符串格式。在调试时很有用。
- `KeyValuePipe`	
  将 Object 或 Map 转换为键值对数组。
- `LowerCasePipe`	
  把文本转换成全小写形式。
- `PercentPipe`	
  将数字转换为百分比字符串，根据确定组大小和分隔符、小数点字符和其他特定于区域设置的配置的区域设置规则进行格式化。
- `SlicePipe`	
  从一个 Array 或 String 中创建其元素一个新子集（slice）。
- `TitleCasePipe`	
  把文本转换成标题形式。 把每个单词的第一个字母转成大写形式，并把单词的其余部分转成小写形式。 单词之间用任意空白字符进行分隔，比如空格、Tab 或换行符。
- `UpperCasePipe`	
  把文本转换成全大写形式。

> 管道操作符要比三目运算符（?:）的优先级高。
> 
> 由于这种优先级设定，如果你要用管道处理三目元算符的结果，就要把整个表达式包裹在括号中
> 
> `{{ (true ? 'true' : 'false') | uppercase }}`

#### 引用变量
在模板中，使用井号 `#` 来声明一个模板变量。模板变量可以引用
```HTML
<input #phone placeholder="phone number" />
<button type="button" (click)="callPhone(phone.value)">Call</button>
```

- 模板中的DOM 元素
  
  如果在标准的 HTML 标记上声明变量，该变量就会引用该元素。

- 指令或组件
  
  如果在组件上声明变量，该变量就会引用该组件实例。
- 来自 `ng-template` 的 `TemplateRef`
  
  如果你在 `<ng-template>` 元素上声明变量，该变量就会引用一个 `TemplateRef` 实例来代表此模板。
- Web 组件

##### 指定引用的名称

由于标签可能存在多种类型(DOM、组件、指令)，需要指定引用的名称来声明要引用的类型。
``` HTML
<form #itemForm="ngForm" (ngSubmit)="onSubmit(itemForm)">
```
在为引用指定`ngForm`名称，则`itemForm`为`NgForm`类型。如果不指定，则`itemForm`为` HTMLFormElement`类型。

##### 作用域
声明引用变量的模板范围



#### 其他
1. 处于安全考虑，不支持`<script>`标签,会被Angular忽略。
2. 除了HTML,`SVG`也可以作为Angular模板。并像 HTML 模板一样使用指令和绑定



## 样式
### 如何定义
1. 通过`style`指定样式数组,定义一个或者多个内联样式。适合内容简单的组件。
2. 通过`styleUrls`指定CSS文件数组，可以指定多个CSS文件哦！。适合内容比较复杂的组件。
3. 内联在模板的 HTML 中
4. 通过 CSS 文件导入, 可以在styleUrls指定的CSS文件中导入其他CSS文件
   ``` CSS
   /* AOT编译器需要`./`来提示正在使用本地文件 */
   /* 目录相对于当前CSS的目录 */
   @import './path-to-imported-style.css';
   ```
### 样式的范围
通过`@Component`的`encapsulation`选项，可以设定当前组件样式的适用范围。
1. `ViewEncapsulation.ShadowDom` 仅限当前组件，采用浏览器内置API实现
2. `ViewEncapsulation.Emulated` 仅限当前组件，采用添加随机属性实现
3. `ViewEncapsulation.None` 适用于全局
### 支持自定义样式
1. 使用CSS自定义属性
   > **推荐**: 虽然这需要为每个自定义点定义一个自定义属性，但它创建了一个清晰的 API 契约，可以在所有样式的封装模式下工作。
2. 使用 @mixin 声明全局 CSS
3. 使用 CSS `::part` 自定义
4. 提供 TypeScript API
   > **不推荐**: 这种样式 API 的额外 JavaScript 成本会产生比 CSS 高得多的性能成本
### 特殊选择器
- `:host` 把宿主元素作为目标的唯一方式。要把宿主样式作为条件，就要像函数一样把其它选择器放在 `:host` 后面的括号中
  ``` CSS
  :host {
    font-style: italic;
  }

  :host(.active) {
    font-weight: bold;
  }
  ```
- `:host-context` 在当前组件宿主元素的祖先节点中查找 CSS 类，直到文档的根节点为止。
  ``` CSS
  :host-context(.active) {
    font-style: italic;
  }
  ```
  > 它只能与其它选择器组合使用。

## 组件之间的交互
### 输入
通过`@Input()`定义一个标签属性，从父组件中获取设定的标签属性。
``` TypeScript
@Input() childField:string = "default value";
```
在父组件的模板中设定子组件的属性
``` HTML
<child-component [childField]="parentField"></child-component>
```

### 输出
通过`@Output()`定义一个`EventEmitter`类型的属性，可以实现自定义事件，父组件可以通过事件监听获取子组件的数据
``` TypeScript
// 定义事件属性，记住要进行初始化
@Output() myEvent = new EventEmitter<boolean>();
// ...
// 在需要向父组件发送数据时
myEvent.emit(myData)
```
在父组件模板中，使用`$event`变量获取子组件数据
``` HTML
<child-component (myEvent)="myHandle($event)"></child-component>
```

### 本地变量引用
1. 在父组件的模板中，为子组件标签添加`#childId`属性，则在父组件中的模板中可以直接使用childId引用子组件。如以下例子中，父组件在button中响应click事件时，调用子组件的start/stop方法。
``` HTML
<button type="button" (click)="timer.start()">Start</button>
<button type="button" (click)="timer.stop()">Stop</button>
<child-component #timer></child-component>
```
> `只限模板中`,如果像在组件类中访问，需要使用`@ViewChild`
   
2. 在父组件的类中，可以使用标注`@ViewChild`获取子组件，并访问子组件的属性和方法。
``` TypeScript
@ViewChild(ChildComponent)
private child!: ChildComponent;
```
### Service
两个组件也可以使用Service进行交互
- [例子](https://angular.cn/guide/component-interaction#parent-and-children-communicate-using-a-service)


## 内容投影(组件嵌套)

### 单槽内容投影
组件可以从单一来源接受内容。
使用单个`<ng-content>`元素作为插槽。
`<ng-content>` 元素是一个占位符，它不会创建真正的 DOM 元素。
> `<ng-content>` 的那些自定义属性将被忽略。

### 多槽内容投影
组件可以从多个来源接受内容。
使用多个`<ng-content>`元素作为插槽。
通过`select`属性，可以指定标签名、属性、CSS类、`:not`伪类的任意组合。
不带有`select`的`<ng-content>`将作为默认插槽，接收全部未匹配的子组件。
``` HTML
    <ng-content></ng-content>
    <ng-content select="[someTag]"></ng-content>
```

### 条件内容投影（project）
使用条件内容投影的组件仅在满足特定条件时才渲染内容。

消费组件模板
``` HTML
<condition-component>
  <ng-template contentDirective >
    content to be projected.
  </ng-template>
</condition-component>
```

父组件(ConditionComponent)模板
``` HTML
<!--接收默认内容-->
<ng-content></ng-content>
<!--条件渲染内容-->
<div *ngIf="condition" >
    <!--使用 container作为插槽，ngTemplateOutlet属性指定模板，`TemplateRef`类型的值 -->
    <ng-container [ngTemplateOutlet]="content.templateRef"></ng-container>
</div>
```

父组件(ConditionComponent)类
``` TypeScript
@Component({
  selector: 'condition-component',
  templateUrl: 'condition-component.template.html',
})
export class ConditionComponent {
  @Input() condition = false;
  @ContentChild(ContentDirective) content!: ContentDirective;
}
```

模板内容获取指令(ContentDirective)
``` TypeScript
@Directive({
  selector: '[contentDirective]'
})
export class ContentDirective {
  constructor(public templateRef: TemplateRef<unknown>) {}
}
```


通过把一个指令放在 `<ng-template>` 元素（或一个带 * 前缀的指令）上，可以访问 `TemplateRef `的实例。内嵌视图的 `TemplateRef` 实例会以 `TemplateRef` 作为令牌，注入到该指令的构造函数中。

被插入内容使用`<ng-template>`, 因为在显式渲染`<ng-template>`元素之前，Angular 不会初始化该元素的内容。

> 此类情况下不要使用`<ng-content>`,因为即使不显示，仍会被初始化。



## 动态组件
通过`ViewContainerRef.createComponent`接口可以实现动态组件的创建。
``` TypeScript
viewContainerRef.createComponent<DynamicComponent>(MyDymaticComponent);
```
例子中，`DynamicComponent`未动态组件的接口类，`MyDymaticComponent`是实现了`DynamicComponent`接口的组件类。

通常我们使用`<ng-template>` 元素作为动态加载组件，因为它不会渲染任何额外的输出。
``` html
<ng-template dynamicHost></ng-template>
```
自定义一个包含ViewContainerRef属性的指令(`Directive`)作为动态加载组件的标识。
``` TypeScript
import { Directive, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[dynamicHost]',
})
export class DynamicDirective {
  constructor(public viewContainerRef: ViewContainerRef) { }
}
```
这样就可以在父组件中获取动态加载组件，并使用其`viewContainerRef`属性动态创建子组件。
``` TypeScript
@ViewChild(DynamicDirective, {static: true}) dynamicHost!: DynamicDirective;
// ...
const viewContainerRef = this.dynamicHost.viewContainerRef;
viewContainerRef.clear();
viewContainerRef.createComponent<DynamicComponent>
```

# 指令(Direction)
指令是为 Angular 应用程序中的元素添加额外行为的类。

使用 Angular 的内置指令，可以管理表单、列表、样式以及要让用户看到的任何内容。

## 内置指令
### 内置属性型指令
- NgClass
  
  添加和删除**一组** CSS 类。
  - 接收值为`字符串`的表达式
  - 接收值为`字符串数组`的表达式
  - 接收值为`Record<string,boolean>`的表达式

  > 要添加或删除单个类，请使用类绑定而不是 NgClass。


  TODO 对于多个类，类属性和类指令都可以实现类似效果。有什么区别？


- NgStyle
  
  添加和删除**一组** HTML 样式。
  - 接收值为`Record<string,boolean>`的表达式
  
  > 要添加或删除单个样式，请使用样式绑定而不是 NgClass。

  TODO 对于多个样式，样式属性和样式指令都可以实现类似效果。有什么区别？

- NgModel
  
  将双向数据绑定添加到 HTML 表单元素。
  ``` HTML
  <input [(ngModel)]="currentItem.name" id="example-ngModel">
  ```
  等价于
  ``` HTML
  <input [ngModel]="currentItem.name" (ngModelChange)="currentItem.name=value" id="example-uppercase">
  ```
  NgModel 指令适用于ControlValueAccessor支持的元素。Angular 为所有基本 HTML 表单元素提供了值访问器。

  > **与双向绑定区别** 除了绑定input的value值之外，还可以处理radio, checkbox, select等复杂Form元素。

### 内置结构型指令
- NgIf

  有条件地从模板创建或销毁子视图。

  如果 `NgIf` 为 `false`，则 Angular 将从 DOM 中移除一个元素及其后代。然后，Angular 会销毁其组件，从而释放内存和资源。
  ``` HTML
  <!--常用case: 在对象为null时不创建元素-->
  <div *ngIf="currentCustomer">Hello, {{currentCustomer.name}}</div>
  ```

- NgFor

  为列表中的每个条目重复渲染一个节点。
  ``` HTML
  <!--遍历items，当前值为item, 索引为i, 变更检测为trackByItems, 索引和变更检测为可选属性 -->
  <div *ngFor="let item of items; let i=index; trackBy=trackByItems">{{i + 1}} - {{item.name}}</div>
  ```
  其中索引检测的实现例子如下：仅当id发生变化的时候重新渲染当前元素 
  ``` TypeScript
  trackByItems(index: number, item: Item): number { return item.id; }
  ```
- NgSwitch

  一组在备用视图之间切换的指令。
  ``` HTML
  <div [ngSwitch]="currentItem.feature">
    <app-stout-item    *ngSwitchCase="'stout'"    [item]="currentItem"></app-stout-item>
    <app-device-item   *ngSwitchCase="'slim'"     [item]="currentItem"></app-device-item>
    <app-lost-item     *ngSwitchCase="'vintage'"  [item]="currentItem"></app-lost-item>
    <app-best-item     *ngSwitchCase="'bright'"   [item]="currentItem"></app-best-item>
    <!-- . . . -->
    <app-unknown-item  *ngSwitchDefault           [item]="currentItem"></app-unknown-item>
  </div>
  ```

## 属性型指令
使用属性型指令，可以更改 DOM 元素和 Angular 组件的外观或行为。
- 更改DOM元素
  
  可以在指令的 `constructor()` 中添加 `ElementRef` 以注入对宿主 DOM 元素的引用。ElementRef 的 nativeElement 属性会提供对宿主 DOM 元素的直接访问权限。

- 处理用户事件
  
  适用 `@HostListener()` 装饰器可以监听宿主组件的事件。
  ``` TypeScript
  @HostListener('clic') onClick() {
    // do something here
  }
  ```

- 获取绑定值
  
  与组件的值绑定一样，使用`@Input`装饰器定义值绑定。如果指令仅接收一个值，也可以定义与指令名称相同的属性，在适用指令的同时，获取用户的绑定值。假设指令名称为myDirective。
  ``` HTML
  <p [myDirective]="hostField" anotherInput="plainValue">
    Highlight me too!
  </p>
  ```

- 通过 `NgNonBindable` 停用 Angular 处理过程

  例子中，`{{ 1 + 1 }}`将作为字符串显示，而不是2
  ``` HTML
  <p ngNonBindable>This should not evaluate: {{ 1 + 1 }}</p>
  ```

## 结构型指令
结构指令是通过添加和删除 DOM 元素来更改 DOM 布局的指令。

### 结构型指令简写形式
应用结构指令时，它们通常以星号 * 为前缀，例如 *ngIf。
``` HTML
<div *ngIf="hero" class="name">{{hero.name}}</div>
```
是以下形式的语法糖
``` HTML
<ng-template [ngIf]="hero">
  <div class="name">{{hero.name}}</div>
</ng-template>
```
Angular 会将结构指令前面的星号(`*`)转换为围绕宿主元素及其后代的 `<ng-template>`。

`*ngFor` 中的星号的简写形式与非简写的 `<ng-template>` 形式进行比较：
``` HTML

<div
  *ngFor="let hero of heroes; let i=index; let odd=odd; trackBy: trackById"
  [class.odd]="odd">
  ({{i}}) {{hero.name}}
</div>
<!-- 等价于 -->
<ng-template ngFor let-hero [ngForOf]="heroes"
  let-i="index" let-odd="odd" [ngForTrackBy]="trackById">
  <div [class.odd]="odd">
    ({{i}}) {{hero.name}}
  </div>
</ng-template>
```

### 每个元素一个结构指令
只能将一个 结构指令 应用于一个元素。

如果希望在同一个元素上使用多个指令，比如同时使用`*ngIf`和`*ngFor`,以下写法是非法的:
``` HTML
<div *ngIf="condition" *ngFor="let i of list">..</div>
```
可以使用`<ng-container>`作为包装结构实现:
``` HTML
<ng-container *ngIf="condition">
    <div *ngFor="let i of list">..</div>
</ng-container>
```

### 自定义结构指令
1.  在指令的构造函数中将 `TemplateRef` 和 `ViewContainerRef` 注入成私有变量。
2. `TemplateRef`可帮助你获取 `<ng-template>` 的内容，而 `ViewContainerRef` 可以访问视图容器。
3. 添加一个带 `setter` 的 `@Input()` 属性，并实现逻辑

以下例子中为通过`appUnless`属性，控制是否显示`<ng-template>` 的内容
``` TypeScript
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

/**
 * Add the template content to the DOM unless the condition is true.
 */
@Directive({ selector: '[appUnless]'})
export class UnlessDirective {
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef) { }

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}
```

## 指令复用
指令组合 API 提供了一种封装可复用行为的好方法

### 宿主指令(将指令添加到组件)
从组件的 TypeScript 类内部将指令应用于组件的宿主元素。

#### 语法
可以通过将 `hostDirectives` 属性添加到组件的装饰器来将指令应用于组件
``` TypeScript
@Component({
  selector: 'sample-component',
  template: 'sample-component.html',
  hostDirectives: [MyDirective],
})
export class SampeComponent { }
```
#### 效果
- 当框架渲染组件时，Angular 还会创建每个宿主指令的实例。
- 指令的宿主绑定被应用于组件的宿主元素。
- 默认情况下，宿主指令的输入和输出不会作为组件公共 API 的一部分公开。
  
  可以通过扩展 `hostDirectives` 中的条目来在组件的 API 中显式包含输入和输出
  ``` TypeScript
  @Component({
    selector: 'sample-component',
    template: 'sample-component.html',
    hostDirectives: [{
        directive: MyDirective,
        inputs: ['myInput'],
        outputs: ['myOutput'],
    }],
  })
  export class SampeComponent { }
  ```
  也可以为 `hostDirective` 的输入和输出起别名来自定义组件的 API：
  ``` TypeScript
  @Component({
    selector: 'sample-component',
    template: 'sample-component.html',
    hostDirectives: [{
        directive: MyDirective,
        inputs: ['myInput: mi'],
        outputs: ['myOutput: mo'],
    }],
  })
  export class SampeComponent { }
  ```
- Angular 会在编译时静态应用宿主指令。你不能在运行时动态添加指令。
- `hostDirectives` 中使用的指令必须是 `standalone: true` 的。
- Angular 会忽略 `hostDirectives` 属性中所应用的那些指令的 `selector。`

### 将指令添加到指令

#### 语法
``` TypeScript
@Directive({
  hostDirectives: [DirectiveA, DirectiveB],
})
export class ComposeDirective { }
```
#### 效果
- 将组合指令应用与组件时，会创建被组合的各个指令的实例。
- 被组合指令的宿主绑定都会应用于组合指令的宿主元素。

### 指令的执行顺序
- `宿主指令`和`直接在模板中使用的组件和指令`会经历**相同**的生命周期。
- `宿主指令`总是会在应用它们的`组件或指令之前`执行它们的构造函数、生命周期钩子和绑定。

> 这种操作顺序意味着带有 `hostDirectives` 的组件可以改写（`override`）宿主指令指定的任何宿主绑定。

### 指令的依赖注入
- 指定了 `hostDirectives` 的组件或指令可以注入这些宿主指令的实例
- 宿主指令 可以注入组件或指令的实例
- 当把宿主指令应用于组件时，组件和宿主指令都可以定义提供者。
- 如果带有 `hostDirectives` 的组件或指令以及这些宿主指令都提供相同的注入令牌，则带有 `hostDirectives` 的类定义的提供者会优先于宿主指令定义的提供者。

TODO 太抽象，需要例子

# 缺点
1. TODO redux替代品
2. 限定TypeScript
3. 编译巨慢，开发的时候Hot Reload更是要命。

# 常用命令
ng generate module MyModule
ng generate component --module my-module MyComponent
ng generate interface MyData
ng generate service MyService