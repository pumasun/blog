---
title: Markdown速查手册
date: 2023-06-02 13:57:49
tags: ['IT', 'markdown', 'md']
description: 'Markdown速查手册'
keywords: ['markdown', 'Cheat Sheet']
category: ['IT']
---

# Markdown速查手册
[参考](https://www.markdownguide.org/cheat-sheet/)

## 基本语法
<table class="table table-bordered">
  <thead class="thead-light">
    <tr>
      <th>元素</th>
      <th>Markdown语法</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>标题</td>
      <td><code># H1<br>
          ## H2<br>
          ### H3</code></td>
    </tr>
    <tr>
      <td>粗体</td>
      <td><code>**bold text**</code></td>
    </tr>
    <tr>
      <td>斜体</td>
      <td><code>*italicized text*</code></td>
    </tr>
    <tr>
      <td>块引用</td>
      <td><code>&gt; blockquote</code></td>
    </tr>
    <tr>
      <td>有序列表</td>
      <td><code>
        1. First item<br>
        2. Second item<br>
        3. Third item<br>
      </code></td>
    </tr>
    <tr>
      <td>无序列表</td>
      <td>
        <code>
          - First item<br>
          - Second item<br>
          - Third item<br>
        </code>
      </td>
    </tr>
    <tr>
      <td>代码块</td>
      <td><code>`code`</code></td>
    </tr>
    <tr>
      <td>水平线</td>
      <td><code>---</code></td>
    </tr>
    <tr>
      <td>链接</td>
      <td><code>[title](https://www.example.com)</code></td>
    </tr>
    <tr>
      <td>图片</td>
      <td><code>![alt text](image.jpg)</code></td>
    </tr>
  </tbody>
</table>

## 扩展语法
<table class="table table-bordered">
  <thead class="thead-light">
    <tr>
      <th>元素</th>
      <th>Markdown语法</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>表格</td>
      <td><code>
          | Syntax      | Description |<br>
          | ----------- | ----------- |<br>
          | Header      | Title       |<br>
          | Paragraph   | Text        |
      </code></td>
    </tr>
    <tr>
      <td>独立代码块</td>
      <td><code>```<br>
      {<br>
      &nbsp;&nbsp;"firstName": "John",<br>
      &nbsp;&nbsp;"lastName": "Smith",<br>
      &nbsp;&nbsp;"age": 25<br>
      }<br>
      ```
      </code></td>
    </tr>
    <tr>
      <td>脚注</td>
      <td><code>
        Here's a sentence with a footnote. [^1]<br><br>
        [^1]: This is the footnote.
      </code></td>
    </tr>
    <tr>
      <td>标题ID</td>
      <td><code>### My Great Heading {<b>#heading-id</b>}</code></td>
    </tr>
    <tr>
      <td>定义列表</td>
      <td><code>
        term<br>
        : definition
      </code></td>
    </tr>
    <tr>
      <td>Strikethrough</td>
      <td><code>~~The world is flat.~~</code></td>
    </tr>
    <tr>
      <td>任务列表</td>
      <td><code>
        - [x] Write the press release<br>
        - [ ] Update the website<br>
        - [ ] Contact the media
      </code></td>
    </tr>
    <tr>
      <td>Emoji</td>
      <td><code>
        That is so funny! :joy:
      </code></td>
    </tr>
    <tr>
      <td>高亮</td>
      <td><code>
        I need to highlight these ==very important words==.
      </code></td>
    </tr>
    <tr>
      <td>Subscript</td>
      <td><code>
        H~2~O
      </code></td>
    </tr>
    <tr>
      <td>上标</td>
      <td><code>
        X^2^
      </code></td>
    </tr>
  </tbody>
</table>