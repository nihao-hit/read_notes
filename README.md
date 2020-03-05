# README
阅读笔记

## Conventional commits

使用Angular conventional commits格式。

### 提交格式

| type(scope): subject | type:改动类型、scope:改动的上下文、subject:改动的简洁描述    |
| -------------------- | ------------------------------------------------------------ |
| Body                 | （可选）介绍改动的原因或改动的详细描述                       |
| Footer               | （可选）介绍改动的影响：如项目剧烈变化、关闭issues、提及contributors。 |

> ```
> git commit -m "fix(core): remove deprecated and defunct wtf* apis" -m "These apis have been deprecated in v8, so they should stick around till v10, but since they are defunct we are removing them early so that they don't take up payload size." -m "PR Close #33949"
> ```

![提交示例](X:\Program\read_notes\images\final-commit-message.png)

### 改动类型

| type                        | description                                                  |
| --------------------------- | ------------------------------------------------------------ |
| :construction_worker: build​ | 与构建系统(涉及脚本、配置或工具)和包依赖项相关的开发更改     |
| :pencil: docs​               | 与项目相关的文档更改,无论是针对最终用户的外部更改(对于库)还是针对开发人员的内部更改。 |
| :green_heart: ci​            | 与持续集成和部署系统相关的开发更改,包括脚本、配置或工具。    |
| :sparkles: feat​             | 与新的向后兼容能力或功能相关的生产更改。                     |
| :bug: fix​                   | 与向后兼容的错误修复相关的生产更改.                          |
| :zap: perf​                  | 与向后兼容性能改进相关的生产更改。                           |
| :recycle: refactor​          | 与修改代码库相关的开发更改，它既没有添加功能，也没有修复bug,例如删除冗余代码、简化代码、重命名变量等等。 |
| :art: style​                 | 与代码基样式相关的开发更改，而不考虑其含义,例如缩进、分号、引号、结尾逗号等等。 |
| :white_check_mark: test​     | 与测试相关的开发更改,例如重构现有测试或添加新测试。          |

### 本书提交格式

由于本项目为纯文档，因此仅使用部分改动类型。

| type          | description                    |
| ------------- | ------------------------------ |
| :pencil: docs​ | 第一版文档更改。               |
| :bug: fix​     | 与发布、格式相关错误的更改。   |
| :zap: perf​    | 可能有的第二版文档更改。       |
| :art: style​   | 文档风格更改，不涉及具体内容。 |

#### 参考文献

https://nitayneeman.com/posts/understanding-semantic-commit-messages-using-git-and-angular/