---
title: "logstash写插件"
author: 一张狗
lastmod: 2018-05-22 12:07:10
date: 2018-02-13 12:00:47
tags: []
---


1. 生成插件模板：

[work@your_machine logstash]$ ./bin/logstash-plugin generate –type input –name demo –path ./plugin-test/

2.目录结构：

[work@your_machine logstash-input-demo]$ tree  
 .  
 ├── CHANGELOG.md  
 ├── CONTRIBUTORS  
 ├── DEVELOPER.md  
 ├── Gemfile  
 ├── lib  
 │   └── logstash  
 │   └── inputs  
 │   └── demo.rb  
 ├── LICENSE  
 ├── logstash-input-demo.gemspec  
 ├── Rakefile  
 ├── README.md  
 └── spec  
 └── inputs  
 └── demo_spec.rb

5 directories, 10 files

 

3.修改

logstash-input-demo.gemspec中todo内容修改

demo.rb中编写代码

4.

执行安装依赖：  
 bundle install

测试插件  
 bundle exec rspec

生成 gem 包  
 /path/to/jruby-veriosn/bin/gem build {logstash-filter-example}.gemspec

安装&查看插件  
 /path/to/logstash-version/bin/logstash-plugin install /path/to/gemfile/path/to/logstash-version/bin/logstash-plugin list

todo:

https://www.jianshu.com/p/756e404735af

https://doc.yonyoucloud.com/doc/logstash-best-practice-cn/dive_into/write_your_own.html


